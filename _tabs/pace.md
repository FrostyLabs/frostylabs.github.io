---
layout: page
title: Pace Converter
---

<style>
.converter { max-width: 540px; margin: 1rem 0 2rem; }
.converter label { display:block; font-weight:600; margin-top:0.75rem; }
.converter input { width:100%; padding:0.6rem; font-size:1rem; }
.converter .hint { font-size:0.9rem; opacity:0.75; }
.output { margin-top:0.75rem; font-weight:600; }
</style>

<div class="converter">
  <p class="hint">Type minutes:seconds or decimal minutes. Examples: <code>5:00</code>, <code>4:30</code>, <code>3.75</code></p>

  <label for="perKm">Pace (min/km)</label>
  <input id="perKm" placeholder="e.g. 5:00">

  <label for="perMi">Pace (min/mi)</label>
  <input id="perMi" placeholder="e.g. 8:03">

  <div class="output" id="example"></div>
</div>

<script>
(() => {
  const KM_PER_MI = 1.609344;

  function parsePace(v) {
    v = (v || "").trim();
    if (!v) return null;
    if (v.includes(":")) {
      const [m, sRaw] = v.split(":");
      const mNum = parseInt(m, 10);
      const sNum = parseFloat(String(sRaw).replace(",", "."));
      if (Number.isNaN(mNum) || Number.isNaN(sNum)) return null;
      return mNum * 60 + sNum;
    } else {
      const dec = parseFloat(v.replace(",", "."));
      if (Number.isNaN(dec)) return null;
      return dec * 60;
    }
  }

  function formatPace(totalSeconds) {
    if (totalSeconds == null || !isFinite(totalSeconds)) return "";
    let s = Math.round(totalSeconds);
    const m = Math.floor(s / 60);
    s = s - m * 60;
    return `${m}:${String(s).padStart(2, "0")}`;
  }

  function initConverter() {
    const km = document.querySelector("#perKm");
    const mi = document.querySelector("#perMi");
    const ex = document.querySelector("#example");
    if (!km || !mi || !ex) return;

    let updating = false;

    function syncFromKm() {
      if (updating) return;
      const secKm = parsePace(km.value);
      if (secKm == null) { mi.value = ""; ex.textContent = ""; return; }
      const secMi = secKm * KM_PER_MI;
      updating = true;
      mi.value = formatPace(secMi);
      ex.textContent = `Example: ${km.value} min/km ≈ ${mi.value} min/mi`;
      updating = false;
    }

    function syncFromMi() {
      if (updating) return;
      const secMi = parsePace(mi.value);
      if (secMi == null) { km.value = ""; ex.textContent = ""; return; }
      const secKm = secMi / KM_PER_MI;
      updating = true;
      km.value = formatPace(secKm);
      ex.textContent = `Example: ${km.value} min/km ≈ ${mi.value} min/mi`;
      updating = false;
    }

    km.removeEventListener("input", syncFromKm);
    mi.removeEventListener("input", syncFromMi);
    km.addEventListener("input", syncFromKm);
    mi.addEventListener("input", syncFromMi);

    if (!km.value) {
      km.value = "5:00";
      syncFromKm();
    } else {
      syncFromKm();
    }
  }

  // Run on normal loads…
  document.addEventListener("DOMContentLoaded", initConverter);
  // …and also after PJAX/AJAX navigation.
  ["pjax:complete", "pjax:success", "pjax:end"].forEach(evt =>
    document.addEventListener(evt, initConverter)
  );
})();
</script>
