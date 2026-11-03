
<script lang="ts">
  import { onMount } from "svelte";

  // ====================================
  // 1. UNTERNEHMENSDATEN
  // ====================================

  let unternehmen = $state("Beispielunternehmen GmbH");
  let analyseName = $state("Profitabilitätsanalyse");

  let preis = $state(50);
  let variableKosten = $state(30);
  let fixkosten = $state(20000);
  let menge = $state(1500);

  let szenario = $state<"worst" | "base" | "best">("base");

  type Analyse = {
    id: number;
    unternehmen: string;
    analyseName: string;
    preis: number;
    variableKosten: number;
    fixkosten: number;
    menge: number;
  };

  let analysen = $state<Analyse[]>([]);
  let aktiveId = $state(0);
  let geladen = $state(false);

  // ====================================
  // 2. FINANZBERECHNUNGEN
  // ====================================

  function berechnen(
    preisFaktor: number,
    kostenFaktor: number,
    mengenFaktor: number
  ) {
    const p = preis * preisFaktor;
    const k = variableKosten * kostenFaktor;
    const m = Math.round(menge * mengenFaktor);

    const umsatz = p * m;
    const kosten = fixkosten + k * m;
    const gewinn = umsatz - kosten;
    const deckungsbeitrag = p - k;

    const breakEven =
      deckungsbeitrag > 0
        ? fixkosten / deckungsbeitrag
        : null;

    const marge =
      umsatz > 0 ? (gewinn / umsatz) * 100 : 0;

    return {
      preis: p,
      kostenProStueck: k,
      menge: m,
      umsatz,
      kosten,
      gewinn,
      deckungsbeitrag,
      breakEven,
      marge
    };
  }

  // Szenarien
  let base = $derived(berechnen(1, 1, 1));

  let worst = $derived(
    berechnen(0.95, 1.10, 0.80)
  );

  let best = $derived(
    berechnen(1.05, 0.95, 1.20)
  );

  // Ausgewähltes Szenario
  let aktuell = $derived(
    szenario === "worst"
      ? worst
      : szenario === "best"
        ? best
        : base
  );

  // Gültige Eingaben
  let gueltig = $derived(
    unternehmen.trim().length > 0 &&
    analyseName.trim().length > 0 &&
    [preis, variableKosten, fixkosten, menge].every(
      wert =>
        Number.isFinite(wert) &&
        wert >= 0 &&
        wert <= 1000000000
    ) &&
    Number.isInteger(menge)
  );

  // ====================================
  // 3. FORMATIERUNG
  // ====================================

  function euro(wert: number) {
    return new Intl.NumberFormat("de-AT", {
      style: "currency",
      currency: "EUR"
    }).format(wert);
  }

  function zahl(wert: number) {
    return Math.round(wert).toLocaleString("de-AT");
  }

  function prozent(wert: number) {
    return wert.toFixed(1).replace(".", ",") + " %";
  }

  // ====================================
  // 4. LOCAL STORAGE
  // ====================================

  const SPEICHER = "profitcompass-einfach-v1";
  const LISTE = "profitcompass-analysen-v1";

  onMount(() => {
    try {
      const daten = localStorage.getItem(SPEICHER);

      if (daten) {
        const d = JSON.parse(daten);

        if (typeof d.unternehmen === "string")
          unternehmen = d.unternehmen;

        if (typeof d.analyseName === "string")
          analyseName = d.analyseName;

        if (typeof d.preis === "number")
          preis = d.preis;

        if (typeof d.variableKosten === "number")
          variableKosten = d.variableKosten;

        if (typeof d.fixkosten === "number")
          fixkosten = d.fixkosten;

        if (typeof d.menge === "number")
          menge = d.menge;
      }

      const liste = localStorage.getItem(LISTE);

      if (liste) {
        const daten = JSON.parse(liste);

        if (Array.isArray(daten)) {
          analysen = daten;
        }
      }
    } catch (error) {
      console.error("Fehler beim Laden:", error);
    }

    geladen = true;
  });

  // Eingaben automatisch speichern
  $effect(() => {
    if (!geladen || !gueltig) return;

    try {
      localStorage.setItem(
        SPEICHER,
        JSON.stringify({
          unternehmen,
          analyseName,
          preis,
          variableKosten,
          fixkosten,
          menge
        })
      );

      localStorage.setItem(
        LISTE,
        JSON.stringify(analysen)
      );
    } catch (error) {
      console.error("Speichern fehlgeschlagen:", error);
    }
  });

  // ====================================
  // 5. ANALYSEN SPEICHERN
  // ====================================

  function speichern() {
    if (!gueltig) return;

    const id = aktiveId || Date.now();

    const neueAnalyse: Analyse = {
      id,
      unternehmen,
      analyseName,
      preis,
      variableKosten,
      fixkosten,
      menge
    };

    analysen = [
      neueAnalyse,
      ...analysen.filter(a => a.id !== id)
    ];

    aktiveId = id;
  }

  function laden(analyse: Analyse) {
    aktiveId = analyse.id;
    unternehmen = analyse.unternehmen;
    analyseName = analyse.analyseName;
    preis = analyse.preis;
    variableKosten = analyse.variableKosten;
    fixkosten = analyse.fixkosten;
    menge = analyse.menge;
    szenario = "base";
  }

  function neueAnalyse() {
    aktiveId = 0;
    analyseName = "Neue Analyse";
    unternehmen = "Neues Unternehmen";
    preis = 50;
    variableKosten = 30;
    fixkosten = 20000;
    menge = 1500;
    szenario = "base";
  }

  function loeschen(id: number) {
    analysen = analysen.filter(a => a.id !== id);

    if (aktiveId === id) {
      aktiveId = 0;
    }
  }

  // ====================================
  // 6. MANAGEMENT-EMPFEHLUNGEN
  // ====================================

  let bewertung = $derived.by(() => {
    if (base.deckungsbeitrag <= 0) {
      return "Kritische Preis- und Kostenstruktur";
    }

    if (base.gewinn < 0) {
      return "Optimierungsbedarf";
    }

    if (base.gewinn === 0) {
      return "Break-even erreicht";
    }

    if (worst.gewinn < 0) {
      return "Profitabel mit Szenariorisiko";
    }

    return "Profitables Geschäftsmodell";
  });

  let empfehlungen = $derived.by(() => {
    if (base.deckungsbeitrag <= 0) {
      return [
        "Verkaufspreis und variable Stückkosten überprüfen.",
        "Einkaufs- und Produktionskosten reduzieren.",
        "Vor einer Absatzsteigerung einen positiven Deckungsbeitrag erzielen.",
        "Das Geschäftsmodell nach den Anpassungen neu kalkulieren."
      ];
    }

    if (base.gewinn < 0) {
      return [
        `Den Absatz in Richtung der Break-even-Menge von ${zahl(Math.ceil(base.breakEven ?? 0))} Stück erhöhen.`,
        "Preisstrategie und mögliche Preisanpassungen prüfen.",
        "Fixkosten auf Einsparungspotenziale untersuchen.",
        "Monatliches Controlling von Umsatz und Deckungsbeitrag einführen."
      ];
    }

    if (base.gewinn === 0) {
      return [
        "Einen positiven Gewinn und finanziellen Sicherheitspuffer aufbauen.",
        "Den Deckungsbeitrag je Stück verbessern.",
        "Kosten- und Absatzschwankungen regelmäßig kontrollieren."
      ];
    }

    return [
      "Die bestehende Profitabilität durch regelmäßiges Finanzcontrolling sichern.",
      "Den Deckungsbeitrag und die Entwicklung der variablen Kosten überwachen.",
      "Wachstumspotenziale unter Berücksichtigung der zusätzlichen Kosten prüfen.",
      "Worst-Case-Szenarien als Frühwarnsystem im Management verwenden.",
      "Vor größeren Investitionen eine aktualisierte Wirtschaftlichkeitsrechnung erstellen."
    ];
  });

  // ====================================
  // 7. BREAK-EVEN-DIAGRAMM
  // ====================================

  function diagramm(daten: typeof base) {
    const breite = 600;
    const links = 60;
    const rechts = 20;
    const oben = 20;
    const unten = 225;

    const maxMenge = Math.max(
      10,
      daten.menge * 1.4,
      (daten.breakEven ?? 0) * 1.2
    );

    const maxWert = Math.max(
      1,
      daten.preis * maxMenge,
      fixkosten + daten.kostenProStueck * maxMenge
    ) * 1.1;

    const x = (menge: number) =>
      links + (menge / maxMenge) * (breite - links - rechts);

    const y = (wert: number) =>
      unten - (wert / maxWert) * (unten - oben);

    return {
      umsatzLinie:
        `M ${x(0)} ${y(0)} L ${x(maxMenge)} ${y(daten.preis * maxMenge)}`,

      kostenLinie:
        `M ${x(0)} ${y(fixkosten)} L ${x(maxMenge)} ${y(fixkosten + daten.kostenProStueck * maxMenge)}`,

      planX: x(daten.menge),

      breakEvenX:
        daten.breakEven === null ? null : x(daten.breakEven),

      breakEvenY:
        daten.breakEven === null
          ? null
          : y(daten.preis * daten.breakEven),

      maxMenge,
      maxWert,
      links,
      rechts,
      oben,
      unten
    };
  }

  let chart = $derived(diagramm(aktuell));
  let reportChart = $derived(diagramm(base));

  // ====================================
  // 8. PDF-REPORT
  // ====================================

  const berichtsdatum = new Date().toLocaleDateString("de-AT");

  function pdfExport() {
    if (!gueltig) return;

    // Öffnet nur den speziell gestalteten PDF-Report
    window.print();
  }
</script>


<svelte:head>
  <title>ProfitCompass | .Mittersteiner Consulting</title>
</svelte:head>


<!-- ====================================
     WEB-DASHBOARD
     ==================================== -->

<div class="app">

  <header class="header">
    <div>
      <div class="brand">
        <span class="brand-dot">.</span>Mittersteiner
        <span class="brand-sub">CONSULTING</span>
      </div>

      <h1>ProfitCompass</h1>

      <p>Financial Analysis & Management Advisory</p>
    </div>

    <div class="header-buttons">
      <button class="button-secondary" onclick={speichern} disabled={!gueltig}>
        Analyse speichern
      </button>

      <button class="button-primary" onclick={pdfExport} disabled={!gueltig}>
        PDF-Report erstellen
      </button>
    </div>
  </header>


  <!-- UNTERNEHMENSDATEN -->

  <section class="panel">
    <div class="section-header">
      <div>
        <p class="eyebrow">01 / DATENGRUNDLAGE</p>
        <h2>Unternehmensdaten</h2>
      </div>

      <span class="save-label">● Automatische Speicherung</span>
    </div>

    <div class="grid-2">
      <label>
        Unternehmensname
        <input type="text" bind:value={unternehmen} />
      </label>

      <label>
        Analysebezeichnung
        <input type="text" bind:value={analyseName} />
      </label>
    </div>

    <div class="grid-4">
      <label>
        Verkaufspreis / Stück (€)
        <input type="number" min="0" bind:value={preis} />
      </label>

      <label>
        Variable Kosten / Stück (€)
        <input type="number" min="0" bind:value={variableKosten} />
      </label>

      <label>
        Fixkosten (€)
        <input type="number" min="0" bind:value={fixkosten} />
      </label>

      <label>
        Geplanter Absatz
        <input type="number" min="0" step="1" bind:value={menge} />
      </label>
    </div>

    {#if !gueltig}
      <p class="error">
        Bitte gültige, nicht negative Zahlen eingeben.
        Die Absatzmenge muss eine ganze Zahl sein.
      </p>
    {/if}
  </section>


  {#if gueltig}

    <!-- KENNZAHLEN -->

    <section class="kpis">

      <div class="kpi">
        <span>Umsatz</span>
        <strong>{euro(base.umsatz)}</strong>
        <small>Base Case</small>
      </div>

      <div class="kpi">
        <span>Gewinn / Verlust</span>
        <strong class:negative={base.gewinn < 0}>
          {euro(base.gewinn)}
        </strong>
        <small>Base Case</small>
      </div>

      <div class="kpi">
        <span>Break-even</span>
        <strong>
          {base.breakEven === null
            ? "Nicht erreichbar"
            : `${zahl(Math.ceil(base.breakEven))} Stück`}
        </strong>
        <small>Gewinnschwelle</small>
      </div>

      <div class="kpi">
        <span>Gewinnmarge</span>
        <strong>{prozent(base.marge)}</strong>
        <small>Base Case</small>
      </div>

    </section>


    <!-- DIAGRAMM -->

    <section class="panel">

      <div class="section-header">
        <div>
          <p class="eyebrow">02 / BREAK-EVEN-ANALYSE</p>
          <h2>Umsatz und Gesamtkosten</h2>
          <p>Vergleich der verschiedenen Szenarien</p>
        </div>
      </div>

      <div class="scenario-buttons">
        <button
          class:active={szenario === "worst"}
          onclick={() => szenario = "worst"}
        >
          Worst Case
        </button>

        <button
          class:active={szenario === "base"}
          onclick={() => szenario = "base"}
        >
          Base Case
        </button>

        <button
          class:active={szenario === "best"}
          onclick={() => szenario = "best"}
        >
          Best Case
        </button>
      </div>

      <div class="scenario-summary">
        <div>
          <span>Verkaufspreis</span>
          <strong>{euro(aktuell.preis)}</strong>
        </div>

        <div>
          <span>Absatzmenge</span>
          <strong>{zahl(aktuell.menge)} Stück</strong>
        </div>

        <div>
          <span>Ergebnis</span>
          <strong class:negative={aktuell.gewinn < 0}>
            {euro(aktuell.gewinn)}
          </strong>
        </div>
      </div>

      <div class="chart-container">
        <svg viewBox="0 0 600 260" role="img" aria-label="Break-even-Diagramm">

          <!-- Hilfslinien -->
          {#each [0, 0.25, 0.5, 0.75, 1] as t}
            <line
              x1="60"
              x2="580"
              y1={225 - t * 205}
              y2={225 - t * 205}
              class="gridline"
            />

            <text x="53" y={229 - t * 205} class="axis-label" text-anchor="end">
              {zahl(chart.maxWert * t)}
            </text>

            <text x={60 + t * 520} y="245" class="axis-label" text-anchor="middle">
              {zahl(chart.maxMenge * t)}
            </text>
          {/each}

          <!-- Achsen -->
          <line x1="60" y1="20" x2="60" y2="225" class="axis" />
          <line x1="60" y1="225" x2="580" y2="225" class="axis" />

          <!-- Geplante Absatzmenge -->
          <line
            x1={chart.planX}
            x2={chart.planX}
            y1="20"
            y2="225"
            class="plan-line"
          />

          <!-- Umsatzlinie -->
          <path d={chart.umsatzLinie} class="revenue-line" />

          <!-- Kostenlinie -->
          <path d={chart.kostenLinie} class="cost-line" />

          <!-- Break-even-Punkt -->
          {#if chart.breakEvenX !== null && chart.breakEvenY !== null}
            <circle
              cx={chart.breakEvenX}
              cy={chart.breakEvenY}
              r="6"
              fill="#15947a"
              stroke="white"
              stroke-width="2"
            />
          {/if}
        </svg>
      </div>

      <div class="legend">
        <span><i class="revenue"></i> Umsatz</span>
        <span><i class="costs"></i> Gesamtkosten</span>
        <span><i class="planned"></i> Geplanter Absatz</span>
        <span><i class="break-even"></i> Break-even</span>
      </div>
    </section>


    <!-- SZENARIOVERGLEICH -->

    <section class="panel">

      <p class="eyebrow">03 / SCENARIO ANALYSIS</p>
      <h2>Szenarioanalyse</h2>

      <p class="description">
        Die Szenarien zeigen, wie sich veränderte Preise,
        Kosten und Absatzmengen auf das Ergebnis auswirken.
      </p>

      <div class="scenario-grid">
        <div class="scenario-card">
          <span>Worst Case</span>
          <strong class:negative={worst.gewinn < 0}>
            {euro(worst.gewinn)}
          </strong>
          <small>Preis −5 % · Kosten +10 % · Absatz −20 %</small>
        </div>

        <div class="scenario-card">
          <span>Base Case</span>
          <strong class:negative={base.gewinn < 0}>
            {euro(base.gewinn)}
          </strong>
          <small>Aktuelle Unternehmensplanung</small>
        </div>

        <div class="scenario-card">
          <span>Best Case</span>
          <strong class:negative={best.gewinn < 0}>
            {euro(best.gewinn)}
          </strong>
          <small>Preis +5 % · Kosten −5 % · Absatz +20 %</small>
        </div>
      </div>
    </section>


    <!-- BERATUNG -->

    <section class="panel">

      <p class="eyebrow">04 / MANAGEMENT ADVISORY</p>
      <h2>Strategische Handlungsempfehlung</h2>

      <div class="assessment">{bewertung}</div>

      <p class="description">
        Die folgenden Empfehlungen basieren auf der
        aktuellen wirtschaftlichen Ausgangslage.
      </p>

      <ol class="recommendations">
        {#each empfehlungen as empfehlung}
          <li>{empfehlung}</li>
        {/each}
      </ol>

    </section>

  {/if}


  <!-- GESPEICHERTE ANALYSEN -->

  <section class="panel">

    <div class="section-header">
      <div>
        <p class="eyebrow">05 / WORKSPACE</p>
        <h2>Gespeicherte Unternehmensanalysen</h2>
      </div>

      <button class="button-secondary" onclick={neueAnalyse}>
        Neue Analyse
      </button>
    </div>

    {#if analysen.length === 0}
      <p class="description">
        Noch keine Analysen gespeichert.
      </p>
    {:else}
      <div class="saved-list">
        {#each analysen as analyse}
          <div class="saved-item">
            <div>
              <strong>{analyse.analyseName}</strong>
              <span>{analyse.unternehmen}</span>
            </div>

            <div class="saved-actions">
              <button onclick={() => laden(analyse)}>Öffnen</button>
              <button onclick={() => loeschen(analyse.id)}>Löschen</button>
            </div>
          </div>
        {/each}
      </div>
    {/if}
  </section>


  <footer>
    ProfitCompass © .Mittersteiner Consulting
  </footer>

</div>


<!-- ====================================
     SEPARATER PDF-CONSULTING-REPORT
     Wird nur beim Drucken angezeigt
     ==================================== -->

{#if gueltig}
<div class="report">

  <!-- SEITE 1 -->
  <section class="paper">
    <div class="report-header">
      <strong>.Mittersteiner Consulting</strong>
      <span>FINANCIAL ADVISORY</span>
    </div>

    <div class="report-cover">
      <p>MANAGEMENT CONSULTING REPORT</p>

      <h1>ProfitCompass</h1>

      <h2>{analyseName}</h2>

      <div class="report-divider"></div>

      <p class="report-company">{unternehmen}</p>

      <p>Erstellt am {berichtsdatum}</p>
    </div>

    <div class="report-section">
      <h2>Executive Summary</h2>

      <p>
        Die vorliegende Analyse untersucht die Wirtschaftlichkeit
        des Unternehmens auf Basis der geplanten Absatzmenge,
        des Verkaufspreises sowie der fixen und variablen Kosten.
      </p>

      <p>
        Mit einem erwarteten Umsatz von <strong>{euro(base.umsatz)}</strong>
        und Gesamtkosten von <strong>{euro(base.kosten)}</strong>
        ergibt sich ein geplantes Ergebnis von
        <strong>{euro(base.gewinn)}</strong>.
      </p>

      <p>
        Die Gewinnmarge beträgt <strong>{prozent(base.marge)}</strong>.
        Die wirtschaftliche Ausgangslage wird als
        <strong>{bewertung}</strong> eingestuft.
      </p>
    </div>

    <div class="report-footer">
      .Mittersteiner Consulting | Vertraulich | Seite 1
    </div>
  </section>


  <!-- SEITE 2 -->
  <section class="paper">
    <div class="report-header">
      <strong>.Mittersteiner Consulting</strong>
      <span>FINANCIAL ANALYSIS</span>
    </div>

    <h1>Finanzielle Ausgangslage</h1>

    <table class="report-table">
      <tbody>
        <tr><td>Verkaufspreis / Stück</td><td>{euro(preis)}</td></tr>
        <tr><td>Variable Kosten / Stück</td><td>{euro(variableKosten)}</td></tr>
        <tr><td>Fixkosten</td><td>{euro(fixkosten)}</td></tr>
        <tr><td>Absatzmenge</td><td>{zahl(menge)} Stück</td></tr>
        <tr><td>Umsatz</td><td>{euro(base.umsatz)}</td></tr>
        <tr><td>Gesamtkosten</td><td>{euro(base.kosten)}</td></tr>
        <tr><td>Deckungsbeitrag / Stück</td><td>{euro(base.deckungsbeitrag)}</td></tr>
        <tr><td>Gewinn / Verlust</td><td>{euro(base.gewinn)}</td></tr>
        <tr><td>Gewinnmarge</td><td>{prozent(base.marge)}</td></tr>
        <tr>
          <td>Break-even-Menge</td>
          <td>
            {base.breakEven === null
              ? "Nicht erreichbar"
              : `${zahl(Math.ceil(base.breakEven))} Stück`}
          </td>
        </tr>
      </tbody>
    </table>

    <h2>Break-even-Diagramm</h2>

    <svg viewBox="0 0 600 260" class="report-chart">
      <line x1="60" y1="20" x2="60" y2="225" stroke="#94a3b8"/>
      <line x1="60" y1="225" x2="580" y2="225" stroke="#94a3b8"/>

      <path d={reportChart.umsatzLinie} fill="none" stroke="#2267b4" stroke-width="3"/>
      <path d={reportChart.kostenLinie} fill="none" stroke="#64748b" stroke-width="3"/>

      {#if reportChart.breakEvenX !== null && reportChart.breakEvenY !== null}
        <circle
          cx={reportChart.breakEvenX}
          cy={reportChart.breakEvenY}
          r="6"
          fill="#15947a"
        />
      {/if}

      <text x="80" y="20" font-size="13" fill="#2267b4">Umsatz</text>
      <text x="80" y="40" font-size="13" fill="#64748b">Gesamtkosten</text>
    </svg>

    <p>
      Der Schnittpunkt zwischen Umsatz und Gesamtkosten
      kennzeichnet die Gewinnschwelle. Oberhalb der Gewinnschwelle
      entsteht bei unveränderter Kostenstruktur ein positiver Gewinn.
    </p>

    <div class="report-footer">
      .Mittersteiner Consulting | Vertraulich | Seite 2
    </div>
  </section>


  <!-- SEITE 3 -->
  <section class="paper">
    <div class="report-header">
      <strong>.Mittersteiner Consulting</strong>
      <span>SCENARIO ANALYSIS</span>
    </div>

    <h1>Szenarioanalyse</h1>

    <p>
      Für die Bewertung möglicher Veränderungen werden drei
      wirtschaftliche Szenarien betrachtet.
    </p>

    <table class="report-table">
      <thead>
        <tr>
          <th>Kennzahl</th>
          <th>Worst Case</th>
          <th>Base Case</th>
          <th>Best Case</th>
        </tr>
      </thead>

      <tbody>
        <tr>
          <td>Absatzmenge</td>
          <td>{zahl(worst.menge)}</td>
          <td>{zahl(base.menge)}</td>
          <td>{zahl(best.menge)}</td>
        </tr>
        <tr>
          <td>Umsatz</td>
          <td>{euro(worst.umsatz)}</td>
          <td>{euro(base.umsatz)}</td>
          <td>{euro(best.umsatz)}</td>
        </tr>
        <tr>
          <td>Gesamtkosten</td>
          <td>{euro(worst.kosten)}</td>
          <td>{euro(base.kosten)}</td>
          <td>{euro(best.kosten)}</td>
        </tr>
        <tr>
          <td>Ergebnis</td>
          <td>{euro(worst.gewinn)}</td>
          <td>{euro(base.gewinn)}</td>
          <td>{euro(best.gewinn)}</td>
        </tr>
      </tbody>
    </table>

    <h2>Grafischer Ergebnisvergleich</h2>

    <div class="report-bars">
      {#each [
        { name: "Worst Case", wert: worst.gewinn },
        { name: "Base Case", wert: base.gewinn },
        { name: "Best Case", wert: best.gewinn }
      ] as item}
        <div>
          <span>{item.name}: {euro(item.wert)}</span>
          <div class="report-bar-bg">
            <div
              class:bar-negative={item.wert < 0}
              class="report-bar"
              style:width={`${Math.abs(item.wert) / Math.max(1, Math.abs(worst.gewinn), Math.abs(base.gewinn), Math.abs(best.gewinn)) * 100}%`}
            ></div>
          </div>
        </div>
      {/each}
    </div>

    <h2>Interpretation</h2>

    <p>
      Der Worst Case berücksichtigt einen niedrigeren Verkaufspreis,
      höhere variable Stückkosten und eine geringere Absatzmenge.
      Der Best Case geht von günstigeren wirtschaftlichen
      Rahmenbedingungen aus.
    </p>

    <p>
      Zwischen dem Worst Case und dem Best Case liegt eine
      Ergebnisspanne von
      <strong>{euro(best.gewinn - worst.gewinn)}</strong>.
      Diese Bandbreite verdeutlicht die Sensitivität des
      Geschäftsmodells gegenüber Veränderungen der Planungsannahmen.
    </p>

    <div class="report-footer">
      .Mittersteiner Consulting | Vertraulich | Seite 3
    </div>
  </section>


  <!-- SEITE 4 -->
  <section class="paper">
    <div class="report-header">
      <strong>.Mittersteiner Consulting</strong>
      <span>STRATEGIC ADVISORY</span>
    </div>

    <h1>Strategische Handlungsempfehlung</h1>

    <div class="report-assessment">
      {bewertung}
    </div>

    <p>
      Auf Grundlage der Profitabilitäts- und Szenarioanalyse
      werden folgende Maßnahmen empfohlen:
    </p>

    <ol class="report-recommendations">
      {#each empfehlungen as empfehlung}
        <li>{empfehlung}</li>
      {/each}
    </ol>

    <h2>Management-Prioritäten</h2>

    <p>
      <strong>Kurzfristig (0–30 Tage):</strong>
      Die wichtigsten finanziellen Schwachstellen analysieren
      und konkrete Zielwerte festlegen.
    </p>

    <p>
      <strong>Mittelfristig (31–60 Tage):</strong>
      Geeignete Optimierungsmaßnahmen umsetzen und
      die Entwicklung der Kennzahlen überwachen.
    </p>

    <p>
      <strong>Langfristig (61–90 Tage):</strong>
      Die Unternehmensplanung mit aktualisierten
      Werten neu berechnen und die Strategie anpassen.
    </p>

    <h2>Modellannahmen und Einschränkungen</h2>

    <p>
      Diese Analyse basiert ausschließlich auf den eingegebenen
      Planwerten. Es werden konstante Stückpreise, konstante
      variable Stückkosten und unveränderte Fixkosten angenommen.
      Steuern, Finanzierungskosten, Liquidität, Kapazitätsgrenzen
      und externe Marktinformationen werden nicht berücksichtigt.
    </p>

    <p>
      Die dargestellten Szenarien sind vereinfachte Modellrechnungen
      und keine statistischen Prognosen. Die Empfehlungen sollten
      vor einer realen unternehmerischen Entscheidung anhand
      zusätzlicher Informationen überprüft werden.
    </p>

    <div class="report-footer">
      .Mittersteiner Consulting | Vertraulich | Seite 4
    </div>
  </section>

</div>
{/if}


<style>
  :global(*) {
    box-sizing: border-box;
  }

  :global(body) {
    margin: 0;
    min-width: 320px;
    font-family: Arial, sans-serif;
    background: #f3f6fa;
    color: #1f3045;
    color-scheme: light;
  }

  :global(#app) {
    width: 100%;
    max-width: none;
    margin: 0;
    padding: 0;
  }

  .app {
    max-width: 1350px;
    margin: auto;
    padding: 30px;
  }

  .header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 20px;
    margin-bottom: 30px;
  }

  .brand {
    font-weight: bold;
    font-size: 19px;
    color: #152d46;
  }

  .brand-dot {
    color: #2563eb;
    font-size: 26px;
  }

  .brand-sub {
    margin-left: 8px;
    color: #64748b;
    font-size: 11px;
    letter-spacing: 1px;
  }

  h1 {
    color: #174c8b;
    font-size: 35px;
    margin: 12px 0 5px;
  }

  .header p {
    color: #64748b;
    font-size: 14px;
    margin: 0;
  }

  .header-buttons {
    display: flex;
    gap: 10px;
    flex-wrap: wrap;
  }

  button {
    cursor: pointer;
    font-family: inherit;
  }

  button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .button-primary,
  .button-secondary {
    border-radius: 8px;
    padding: 12px 17px;
    font-size: 13px;
    font-weight: bold;
  }

  .button-primary {
    background: #205baa;
    color: white;
    border: 1px solid #205baa;
  }

  .button-secondary {
    background: white;
    color: #205baa;
    border: 1px solid #cbd9e8;
  }

  .panel {
    background: white;
    padding: 27px;
    margin-bottom: 20px;
    border: 1px solid #e0e7ef;
    border-radius: 12px;
  }

  .section-header {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    gap: 15px;
    margin-bottom: 20px;
  }

  .eyebrow {
    color: #6582a0;
    font-size: 11px;
    font-weight: bold;
    letter-spacing: 1.4px;
    margin: 0 0 9px;
  }

  h2 {
    font-size: 21px;
    color: #152d46;
    margin: 0 0 15px;
  }

  .section-header h2 {
    margin-bottom: 5px;
  }

  .section-header p:not(.eyebrow),
  .description {
    color: #64748b;
    font-size: 14px;
    line-height: 1.7;
  }

  .save-label {
    color: #18866d;
    background: #e8f7f1;
    padding: 9px 12px;
    border-radius: 6px;
    font-size: 11px;
  }

  .grid-2,
  .grid-4 {
    display: grid;
    gap: 20px;
    margin-top: 15px;
  }

  .grid-2 {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }

  .grid-4 {
    grid-template-columns: repeat(4, minmax(0, 1fr));
  }

  label {
    display: block;
    font-weight: bold;
    font-size: 13px;
    color: #43576d;
  }

  input {
    width: 100%;
    padding: 12px;
    margin-top: 8px;
    border: 1px solid #cfddea;
    border-radius: 7px;
    background: #f8fbff;
    color: #1c334a;
    font-size: 15px;
  }

  input:focus {
    outline: 2px solid #b9d5f8;
    border-color: #2563eb;
  }

  .error {
    color: #b91c1c;
    margin-top: 20px;
  }

  .kpis {
    display: grid;
    grid-template-columns: repeat(4, minmax(0, 1fr));
    gap: 16px;
    margin-bottom: 20px;
  }

  .kpi {
    background: white;
    padding: 22px;
    border: 1px solid #e0e7ef;
    border-radius: 12px;
    min-width: 0;
  }

  .kpi span,
  .kpi small {
    display: block;
    color: #64748b;
  }

  .kpi span {
    font-size: 13px;
  }

  .kpi strong {
    display: block;
    font-size: 23px;
    margin: 12px 0;
    color: #183653;
    overflow-wrap: anywhere;
  }

  .kpi small {
    font-size: 11px;
  }

  .negative {
    color: #c03946 !important;
  }

  .scenario-buttons {
    display: flex;
    gap: 8px;
    margin-bottom: 20px;
  }

  .scenario-buttons button {
    border: 1px solid #d6e2ee;
    background: #f5f8fc;
    color: #435a74;
    border-radius: 7px;
    padding: 11px 18px;
    font-weight: bold;
  }

  .scenario-buttons button.active {
    background: #205baa;
    border-color: #205baa;
    color: white;
  }

  .scenario-summary {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    padding: 17px;
    background: #f5f8fc;
    border-radius: 9px;
    gap: 12px;
  }

  .scenario-summary span {
    display: block;
    color: #728298;
    font-size: 12px;
    margin-bottom: 8px;
  }

  .scenario-summary strong {
    font-size: 17px;
    overflow-wrap: anywhere;
  }

  .chart-container {
    margin-top: 20px;
  }

  .chart-container svg {
    width: 100%;
    max-height: 360px;
  }

  .gridline {
    stroke: #e6edf4;
    stroke-width: 1;
  }

  .axis {
    stroke: #94a3b8;
    stroke-width: 1.5;
  }

  .axis-label {
    fill: #64748b;
    font-size: 12px;
  }

  .revenue-line {
    stroke: #2563eb;
    stroke-width: 3;
    fill: none;
  }

  .cost-line {
    stroke: #64748b;
    stroke-width: 3;
    fill: none;
  }

  .plan-line {
    stroke: #d69a3a;
    stroke-width: 2;
    stroke-dasharray: 6 5;
  }

  .legend {
    display: flex;
    gap: 20px;
    flex-wrap: wrap;
    color: #64748b;
    font-size: 12px;
    margin-top: 8px;
  }

  .legend i {
    display: inline-block;
    width: 14px;
    height: 4px;
    vertical-align: middle;
    margin-right: 6px;
  }

  .legend .revenue {
    background: #2563eb;
  }

  .legend .costs {
    background: #64748b;
  }

  .legend .planned {
    background: #d69a3a;
  }

  .legend .break-even {
    background: #15947a;
  }

  .scenario-grid {
    display: grid;
    grid-template-columns: repeat(3, minmax(0, 1fr));
    gap: 15px;
    margin-top: 20px;
  }

  .scenario-card {
    background: #f8fafc;
    border: 1px solid #dce5ef;
    padding: 20px;
    border-radius: 9px;
  }

  .scenario-card span,
  .scenario-card small {
    display: block;
  }

  .scenario-card span {
    font-weight: bold;
    color: #3d536b;
  }

  .scenario-card strong {
    display: block;
    font-size: 23px;
    margin: 15px 0;
    overflow-wrap: anywhere;
  }

  .scenario-card small {
    color: #718198;
    line-height: 1.5;
  }

  .assessment {
    display: inline-block;
    color: #205baa;
    background: #e7f0fb;
    padding: 10px 14px;
    border-radius: 7px;
    font-weight: bold;
    margin: 10px 0;
  }

  .recommendations {
    padding-left: 25px;
    text-align: left;
    line-height: 1.8;
  }

  .recommendations li {
    margin-bottom: 12px;
    padding-left: 5px;
  }

  .saved-list {
    margin-top: 15px;
  }

  .saved-item {
    display: flex;
    justify-content: space-between;
    gap: 15px;
    padding: 17px 0;
    border-bottom: 1px solid #e6edf4;
  }

  .saved-item strong,
  .saved-item span {
    display: block;
  }

  .saved-item span {
    font-size: 13px;
    color: #64748b;
    margin-top: 5px;
  }

  .saved-actions {
    display: flex;
    gap: 8px;
    align-items: center;
  }

  .saved-actions button {
    border: 1px solid #d8e2ee;
    background: white;
    color: #205baa;
    border-radius: 6px;
    padding: 8px 12px;
  }

  footer {
    text-align: center;
    font-size: 12px;
    color: #94a3b8;
    padding: 20px;
  }

  /* REPORT: Hidden in normal browser view */

  .report {
    display: none;
  }

  /* PRINT / PDF STYLING */

  @media print {
    @page {
      size: A4;
      margin: 0;
    }

    :global(body) {
      background: white !important;
      margin: 0;
    }

    .app {
      display: none !important;
    }

    .report {
      display: block !important;
      color: #1f3045;
      -webkit-print-color-adjust: exact;
      print-color-adjust: exact;
    }

    .paper {
      width: 210mm;
      min-height: 297mm;
      padding: 20mm;
      position: relative;
      page-break-after: always;
      break-after: page;
    }

    .paper:last-child {
      page-break-after: auto;
      break-after: auto;
    }

    .report-header {
      display: flex;
      justify-content: space-between;
      border-bottom: 2px solid #205baa;
      padding-bottom: 12px;
      margin-bottom: 30px;
      font-size: 12px;
      color: #174c8b;
    }

    .report-header span {
      font-size: 9px;
      letter-spacing: 1px;
    }

    .report-cover {
      padding-top: 30mm;
      padding-bottom: 15mm;
    }

    .report-cover > p:first-child {
      font-size: 11px;
      letter-spacing: 2px;
      color: #64748b;
    }

    .report-cover h1 {
      font-size: 36px;
      color: #174c8b;
      margin: 20px 0;
    }

    .report-cover h2 {
      font-size: 22px;
    }

    .report-divider {
      height: 3px;
      width: 65px;
      background: #205baa;
      margin: 25px 0;
    }

    .report-company {
      font-size: 18px;
      font-weight: bold;
    }

    .report-section {
      padding-top: 10mm;
    }

    .paper h1 {
      font-size: 24px;
      margin-bottom: 20px;
    }

    .paper h2 {
      font-size: 17px;
      color: #174c8b;
      margin: 22px 0 12px;
    }

    .paper p {
      font-size: 11px;
      line-height: 1.8;
    }

    .report-table {
      width: 100%;
      border-collapse: collapse;
      margin: 15px 0 22px;
      font-size: 10px;
    }

    .report-table th {
      background: #e7f0fb;
      text-align: left;
      padding: 9px;
      border-bottom: 1px solid #cad8e6;
    }

    .report-table td {
      border-bottom: 1px solid #e2e8f0;
      padding: 9px;
    }

    .report-table td:last-child {
      text-align: right;
      font-weight: bold;
    }

    .report-chart {
      width: 100%;
      max-height: 95mm;
    }

    .report-bars > div {
      margin-bottom: 15px;
    }

    .report-bars span {
      font-size: 11px;
      display: block;
      margin-bottom: 6px;
    }

    .report-bar-bg {
      height: 11px;
      background: #e2e8f0;
      border-radius: 4px;
      overflow: hidden;
    }

    .report-bar {
      background: #2563eb;
      height: 100%;
    }

    .report-bar.bar-negative {
      background: #c03946;
    }

    .report-assessment {
      display: inline-block;
      background: #e7f0fb;
      color: #174c8b;
      padding: 11px 15px;
      font-weight: bold;
      font-size: 12px;
    }

    .report-recommendations {
      margin: 25px 0;
      padding-left: 23px;
    }

    .report-recommendations li {
      font-size: 11px;
      line-height: 1.8;
      margin-bottom: 13px;
    }

    .report-footer {
      position: absolute;
      bottom: 15mm;
      left: 20mm;
      right: 20mm;
      border-top: 1px solid #d5e0ea;
      padding-top: 10px;
      font-size: 9px;
      color: #64748b;
    }
  }

  @media (max-width: 900px) {
    .header {
      flex-direction: column;
      align-items: flex-start;
    }

    .grid-4,
    .kpis {
      grid-template-columns: repeat(2, minmax(0, 1fr));
    }
  }

  @media (max-width: 600px) {
    .app {
      padding: 15px;
    }

    .panel {
      padding: 20px;
    }

    .grid-2,
    .grid-4,
    .scenario-grid {
      grid-template-columns: 1fr;
    }

    .scenario-buttons {
      flex-wrap: wrap;
    }

    .scenario-summary {
      grid-template-columns: 1fr;
    }

    .section-header {
      flex-direction: column;
    }

    .saved-item {
      flex-direction: column; 
    }

    h1 {
      font-size: 29px;
    }
  }
</style>
