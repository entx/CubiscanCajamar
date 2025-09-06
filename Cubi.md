// ==UserScript==
// @name         Complete Master Data Task • Auto Input + Confirm + Tricolops Import (PT/EN labels, remap Length↔Height)
// @namespace    http://tampermonkey.net/
// @version      2.8
// @description  Pull Weight/Length/Width/Height from local API (3 decimals), scale dims ÷10, error if 0/no data. Works with EN/PT labels. API Length → Height field; API Height → Length field.
// @match        https://pumabr.logistics.reply.com/mobile/pumabr*
// @grant        GM_xmlhttpRequest
// @grant        GM_notification
// @connect      127.0.0.1
// @connect      localhost
// ==/UserScript==

(function () {
  'use strict';

  // ---- DEFAULTS (used if API unavailable) ----
  let Width   = 120;
  let Weight  = 350;
  let Length  = 200; // API Length (will go into HEIGHT field)
  let Height  = 80;  // API Height (will go into LENGTH field)
  const StdPack = 10;

  // ---- Titles (trigger screen) ----
  const TITLES = [
    'Complete Master Data Task',
    'Concluir Tarefa de Dados Mestres',
  ];

  // ---- Field labels (EN/PT) ----
  const LABELS = {
    OUTER_QTY: ['Outer Qty (pcs)', 'Quantidade externa (pcs)'],
    LENGTH:    ['Size Length (cm)', 'Largura (cm)'],       // UI Length field
    WIDTH:     ['Size Width (cm)',  'Comprimento (cm)'],   // UI Width field
    HEIGHT:    ['Size Height (cm)', 'Altura (cm)'],        // UI Height field
    WEIGHT:    ['Weight (g)',       'Peso (g)'],
  };

  let firedForThisScreen = false;

  // ===== Tricolops API =====
  const endpoints = ["http://127.0.0.1:8080/data", "http://localhost:8080/data"];

  const norm = (s) =>
    (s || '')
      .toString()
      .normalize('NFD')
      .replace(/\p{Diacritic}/gu, '')
      .toLowerCase()
      .replace(/\s+/g, ' ')
      .trim();

  function fmt3(v) {
    const n = parseFloat(v);
    return Number.isFinite(n) ? Number(n.toFixed(3)) : v;
  }
  const g = (obj, k) => obj[k] ?? obj[k.toLowerCase()] ?? obj[k.toUpperCase()];

  function matchesAny(text, options) {
    const t = norm(text);
    return options.some(opt => norm(opt) === t);
  }

  function findRowByAnyLabel(options) {
    const rows = document.querySelectorAll('.row-anomaly');
    for (const row of rows) {
      const labelEl = row.querySelector('.label-anomaly');
      const txt = (labelEl?.textContent || '').trim();
      if (matchesAny(txt, options)) return row;
    }
    return null;
  }

  function isTargetScreen() {
    const h1 = document.querySelector('h1.title.ng-binding');
    if (!h1) return false;
    const text = (h1.textContent || '').trim();
    return matchesAny(text, TITLES);
  }

  function fetchMeasurements(i = 0) {
    return new Promise((resolve) => {
      if (i >= endpoints.length) return resolve(false);

      GM_xmlhttpRequest({
        method: "GET",
        url: endpoints[i],
        timeout: 5000,
        onload: (res) => {
          try {
            const d = JSON.parse(res.responseText || "{}");
            if (!Object.keys(d).length) throw 0;

            // Weight unchanged; dims ÷10
            const newWeight = fmt3(g(d, "Weight"));
            const newLength = fmt3(g(d, "Length") / 10); // will go to HEIGHT field
            const newWidth  = fmt3(g(d, "Width")  / 10); // stays to WIDTH field
            const newHeight = fmt3(g(d, "Height") / 10); // will go to LENGTH field

            // Any dimension 0 → invalid
            if (newLength === 0 || newWidth === 0 || newHeight === 0) return resolve(false);

            if (Number.isFinite(newWeight)) Weight = newWeight;
            if (Number.isFinite(newLength)) Length = newLength;
            if (Number.isFinite(newWidth))  Width  = newWidth;
            if (Number.isFinite(newHeight)) Height = newHeight;

            resolve(true);
          } catch { resolve(false); }
        },
        onerror:  () => resolve(fetchMeasurements(i + 1)),
        ontimeout: () => resolve(fetchMeasurements(i + 1)),
      });
    });
  }

  // ===== DOM helpers =====
  const nativeValueSetter = Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, 'value')?.set;
  function setInputValue(input, value) {
    if (!input) return;
    if (nativeValueSetter) nativeValueSetter.call(input, value);
    else input.value = value;
    input.dispatchEvent(new Event('input',  { bubbles: true }));
    input.dispatchEvent(new Event('change', { bubbles: true }));
  }

  function promptStdPackIfNeeded() {
    const outerQtyRow = findRowByAnyLabel(LABELS.OUTER_QTY);
    if (!outerQtyRow) return null;
    while (true) {
      const raw = window.prompt('Por favor, informe o conteúdo do Standard Pack (número de peças por caixa)');
      if (raw === null) {
        alert('Por favor, informe um valor correto.');
        continue;
      }
      const num = Number(raw);
      if (Number.isInteger(num) && num >= 1 && num <= 200) {
        const input = outerQtyRow.querySelector('input.value-anomaly[type="number"]');
        setInputValue(input, num);
        return num;
      } else {
        alert('Por favor, informe um valor correto.');
      }
    }
  }

  // ===== Fill using remapped rules =====
  function fillDimensionsAndWeight() {
    // WEIGHT row (unchanged)
    const weightRow = findRowByAnyLabel(LABELS.WEIGHT);
    if (weightRow) {
      weightRow.querySelectorAll('input.value-anomaly[type="number"]')
        .forEach(inp => setInputValue(inp, Weight));
    }

    // API Height → UI LENGTH field
    const lengthRow = findRowByAnyLabel(LABELS.LENGTH);
    if (lengthRow) {
      const input = lengthRow.querySelector('input.value-anomaly[type="number"]');
      setInputValue(input, Height);
    }

    // UI WIDTH field stays mapped to API Width
    const widthRow = findRowByAnyLabel(LABELS.WIDTH);
    if (widthRow) {
      const input = widthRow.querySelector('input.value-anomaly[type="number"]');
      setInputValue(input, Width);
    }

    // API Length → UI HEIGHT field
    const heightRow = findRowByAnyLabel(LABELS.HEIGHT);
    if (heightRow) {
      const input = heightRow.querySelector('input.value-anomaly[type="number"]');
      setInputValue(input, Length);
    }
  }

  // Confirm click
  function runConfirm() {
    const btn = document.querySelector('ion-footer-bar .button.button-full.button-positive');
    if (!btn) return;
    const el  = angular.element(btn);
    const sc  = el.scope() || el.isolateScope();
    const $root = angular.element(document.body).injector().get('$rootScope');
    if (sc && typeof sc.confirm === 'function') $root.$applyAsync(() => sc.confirm());
    else el.triggerHandler('click');
  }

  async function fillAll() {
    const ok = await fetchMeasurements();
    if (!ok) {
      try {
        GM_notification({
          title: "Tricolops",
          text: "Medidas no recibidas, por favor revise la conexión con el Cubiscan",
          timeout: 7000
        });
      } catch {}
      return;
    }

    const hasOuter = !!findRowByAnyLabel(LABELS.OUTER_QTY);
    if (hasOuter) promptStdPackIfNeeded();

    fillDimensionsAndWeight();
    setTimeout(runConfirm, 500);
  }

  function onScreenDetected() {
    if (firedForThisScreen) return;
    firedForThisScreen = true;
    setTimeout(fillAll, 2000);
  }

  function check() {
    if (isTargetScreen()) onScreenDetected();
    else firedForThisScreen = false;
  }

  const observer = new MutationObserver(check);
  const startObserver = () => {
    if (!document.body) return;
    observer.observe(document.body, { childList: true, subtree: true });
  };

  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', () => { startObserver(); check(); });
  } else {
    startObserver(); check();
  }

  const origPush = history.pushState;
  const origReplace = history.replaceState;
  history.pushState = function () { const r = origPush.apply(this, arguments); setTimeout(check, 50); return r; };
  history.replaceState = function () { const r = origReplace.apply(this, arguments); setTimeout(check, 50); return r; };
  window.addEventListener('popstate', () => setTimeout(check, 50));
})();
