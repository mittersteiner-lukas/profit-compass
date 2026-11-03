
<script lang="ts">
  import { onMount } from "svelte";

  // 1. Eingabewerte mit Svelte 5
  let verkaufspreis = $state(50);
  let variableKosten = $state(30);
  let fixkosten = $state(20000);
  let menge = $state(1500);

  let geladen = $state(false);
  let speicherFehler = $state(false);

  // Hilfsfunktion: Ist eine Zahl gültig?
  function istGueltig(wert: unknown): wert is number {
    return typeof wert === "number" &&
      Number.isFinite(wert) &&
      wert >= 0;
  }

  // 2. Gespeicherte Eingaben beim Start laden
  onMount(() => {
    try {
      const gespeichert = localStorage.getItem("profitcompass");

      if (gespeichert) {
        const daten = JSON.parse(gespeichert);

        if (istGueltig(daten.verkaufspreis)) {
          verkaufspreis = daten.verkaufspreis;
        }

        if (istGueltig(daten.variableKosten)) {
          variableKosten = daten.variableKosten;
        }

        if (istGueltig(daten.fixkosten)) {
          fixkosten = daten.fixkosten;
        }

        if (
          istGueltig(daten.menge) &&
          Number.isInteger(daten.menge)
        ) {
          menge = daten.menge;
        }
      }
    } catch (fehler) {
      console.error("Laden fehlgeschlagen:", fehler);
    }

    geladen = true;
  });

  // Prüfen, ob alle Eingaben gültig sind
  let eingabenGueltig = $derived(
    istGueltig(verkaufspreis) &&
    istGueltig(variableKosten) &&
    istGueltig(fixkosten) &&
    istGueltig(menge) &&
    Number.isInteger(menge)
  );

  // 3. Daten automatisch im Browser speichern
  $effect(() => {
    // Erst speichern, nachdem alte Daten geladen wurden
    if (!geladen || !eingabenGueltig) return;

    try {
      localStorage.setItem(
        "profitcompass",
        JSON.stringify({
          verkaufspreis,
          variableKosten,
          fixkosten,
          menge
        })
      );

      speicherFehler = false;
    } catch (fehler) {
      speicherFehler = true;
      console.error("Speichern fehlgeschlagen:", fehler);
    }
  });

  // 4. Finanzielle Berechnungen
  let umsatz = $derived(verkaufspreis * menge);

  let deckungsbeitrag = $derived(
    verkaufspreis - variableKosten
  );

  let gesamtkosten = $derived(
    fixkosten + variableKosten * menge
  );

  let gewinn = $derived(
    Math.round((umsatz - gesamtkosten) * 100) / 100
  );

  let breakEven = $derived(
    deckungsbeitrag > 0
      ? Math.ceil(fixkosten / deckungsbeitrag)
      : 0
  );

  let gewinnmarge = $derived(
    umsatz > 0 ? (gewinn / umsatz) * 100 : 0
  );

  // 5. Berechnungen für die Diagramme
  let groessterWert = $derived(
    Math.max(umsatz, gesamtkosten, Math.abs(gewinn), 1)
  );

  let umsatzBreite = $derived(
    (umsatz / groessterWert) * 100
  );

  let kostenBreite = $derived(
    (gesamtkosten / groessterWert) * 100
  );

  let gewinnBreite = $derived(
    (Math.abs(gewinn) / groessterWert) * 100
  );

  // 6. Kennzahlen für die Unternehmensberatung
  let fehlendeMenge = $derived(
    deckungsbeitrag > 0
      ? Math.max(0, breakEven - menge)
      : 0
  );

  let sicherheitspuffer = $derived(
    deckungsbeitrag > 0 && menge > 0
      ? Math.max(
          0,
          ((menge - fixkosten / deckungsbeitrag) / menge) * 100
        )
      : 0
  );

  let notwendigerPreis = $derived(
    menge > 0
      ? variableKosten + fixkosten / menge
      : 0
  );

  // 7. Automatische Kurzbewertung
  let erklaerung = $derived.by(() => {
    if (deckungsbeitrag < 0) {
      return "Der Verkaufspreis liegt unter den variablen Kosten. Jeder zusätzliche Verkauf erhöht den Verlust.";
    }

    if (deckungsbeitrag === 0) {
      return "Das Produkt erzielt keinen positiven Deckungsbeitrag. Zusätzliche Verkäufe verbessern das Ergebnis nicht.";
    }

    if (gewinn < 0) {
      return `Die geplante Absatzmenge liegt unter dem Break-even-Point von ${breakEven} Stück.`;
    }

    if (gewinn === 0) {
      return "Das Unternehmen deckt seine gesamten Kosten, erzielt jedoch noch keinen Gewinn.";
    }

    return `Die geplante Absatzmenge liegt über dem Break-even-Point von ${breakEven} Stück. Das Unternehmen arbeitet profitabel.`;
  });

  // Zahlen als Euro formatieren
  const euroFormat = new Intl.NumberFormat("de-AT", {
    style: "currency",
    currency: "EUR"
  });

  function euro(wert: number) {
    return euroFormat.format(wert);
  }
</script>


<main>
  <div class="container">

    <!-- Firmenkopf -->
    <header>
      <div class="firma">
        <span class="punkt">.</span>Mittersteiner
        <span class="consulting">Consulting</span>
      </div>

      <div class="linie"></div>

      <p class="produkt-label">
        FINANCIAL CONSULTING TOOL
      </p>

      <h1>ProfitCompass</h1>

      <p class="untertitel">
        Profitability & Break-even Analysis
      </p>
    </header>


    <!-- Dashboard -->
    <div class="dashboard">

      <!-- Unternehmensdaten -->
      <section class="karte">
        <h2 class="karten-titel">
          Unternehmensdaten
        </h2>

        <label>
          Verkaufspreis pro Stück (€)
          <input
            type="number"
            min="0"
            step="0.01"
            bind:value={verkaufspreis}
          />
        </label>

        <label>
          Variable Kosten pro Stück (€)
          <input
            type="number"
            min="0"
            step="0.01"
            bind:value={variableKosten}
          />
        </label>

        <label>
          Fixkosten (€)
          <input
            type="number"
            min="0"
            step="0.01"
            bind:value={fixkosten}
          />
        </label>

        <label>
          Erwartete Absatzmenge
          <input
            type="number"
            min="0"
            step="1"
            bind:value={menge}
          />
        </label>

        {#if speicherFehler}
          <p class="speicher-fehler">
            Speichern im Browser ist derzeit nicht möglich.
          </p>
        {:else if !eingabenGueltig}
          <p class="speicher-info">
            Bitte gültige, nicht negative Werte eingeben.
          </p>
        {:else}
          <p class="speicher-info">
            ✓ Eingaben werden automatisch gespeichert
          </p>
        {/if}
      </section>


      {#if eingabenGueltig}

        <!-- Ergebnisse -->
        <section class="karte">
          <h2 class="karten-titel">Ergebnisse</h2>

          <div class="ergebnis">
            <span>Umsatz</span>
            <strong>{euro(umsatz)}</strong>
          </div>

          <div class="ergebnis">
            <span>Gesamtkosten</span>
            <strong>{euro(gesamtkosten)}</strong>
          </div>

          <div class="ergebnis">
            <span>Deckungsbeitrag / Stück</span>
            <strong>{euro(deckungsbeitrag)}</strong>
          </div>

          <div class="ergebnis">
            <span>Break-even-Point</span>

            {#if deckungsbeitrag > 0}
              <strong>{breakEven} Stück</strong>
            {:else if deckungsbeitrag === 0 && fixkosten === 0}
              <strong>0 Stück</strong>
            {:else}
              <strong>Nicht erreichbar</strong>
            {/if}
          </div>

          <div class="ergebnis">
            <span>Gewinnmarge</span>
            <strong>{gewinnmarge.toFixed(2)} %</strong>
          </div>
        </section>


        <!-- Interaktives Diagramm -->
        <section class="karte">
          <h2 class="karten-titel">Financial Overview</h2>

          <div class="diagramm">

            <!-- Umsatz -->
            <div class="diagramm-zeile">
              <div class="diagramm-info">
                <span>Umsatz</span>
                <strong>{euro(umsatz)}</strong>
              </div>

              <div class="balken-hintergrund">
                <div
                  class="balken umsatz"
                  style:width={`${umsatzBreite}%`}
                ></div>
              </div>
            </div>

            <!-- Kosten -->
            <div class="diagramm-zeile">
              <div class="diagramm-info">
                <span>Gesamtkosten</span>
                <strong>{euro(gesamtkosten)}</strong>
              </div>

              <div class="balken-hintergrund">
                <div
                  class="balken kosten"
                  style:width={`${kostenBreite}%`}
                ></div>
              </div>
            </div>

            <!-- Gewinn oder Verlust -->
            <div class="diagramm-zeile">
              <div class="diagramm-info">
                <span>
                  {gewinn >= 0 ? "Gewinn" : "Verlust"}
                </span>

                <strong>
                  {euro(Math.abs(gewinn))}
                </strong>
              </div>

              <div class="balken-hintergrund">
                <div
                  class="balken"
                  class:gewinn={gewinn >= 0}
                  class:verlust={gewinn < 0}
                  style:width={`${gewinnBreite}%`}
                ></div>
              </div>
            </div>

          </div>

          <p class="diagramm-hinweis">
            Live-Analyse basierend auf den Unternehmensdaten
          </p>
        </section>

      {:else}
        <section class="karte fehler-karte">
          <h2>Ungültige Eingabe</h2>
          <p>
            Bitte überprüfe deine Eingaben. Preise und Kosten
            dürfen nicht negativ sein. Die Absatzmenge muss
            eine ganze, nicht negative Zahl sein.
          </p>
        </section>
      {/if}

    </div>


    {#if eingabenGueltig}

      <!-- Erwartetes Ergebnis -->
      <section class="gewinn-box">
        <p class="kleine-ueberschrift">
          ERWARTETES ERGEBNIS
        </p>

        {#if gewinn > 0}
          <h2 class="positiv">
            + {euro(gewinn)}
          </h2>
        {:else if gewinn < 0}
          <h2 class="negativ">
            - {euro(Math.abs(gewinn))}
          </h2>
        {:else}
          <h2>{euro(0)}</h2>
        {/if}

        <p
          class="meldung"
          class:positiv-text={gewinn > 0}
          class:negativ-text={gewinn < 0}
        >
          {erklaerung}
        </p>
      </section>


      <!-- Handlungsempfehlung -->
      <section class="beratung">

        <p class="kleine-ueberschrift">
          .MITTERSTEINER CONSULTING
        </p>

        <h2>Strategische Handlungsempfehlung</h2>


        {#if deckungsbeitrag <= 0}

          <div class="status kritisch">
            Kritischer Handlungsbedarf
          </div>

          <p>
            Die aktuelle Preis- und Kostenstruktur sollte
            grundlegend überprüft werden. Es entsteht kein
            positiver Deckungsbeitrag pro verkauftem Stück.
          </p>

          <h3>Empfohlene Maßnahmen</h3>

          <ol>
            <li>
              <strong>Preisstrategie überarbeiten:</strong>
              Der aktuelle Verkaufspreis von
              {euro(verkaufspreis)} muss überprüft werden.
              Für einen positiven Deckungsbeitrag muss er
              über den variablen Kosten von
              {euro(variableKosten)} liegen.
            </li>

            <li>
              <strong>Variable Kosten reduzieren:</strong>
              Einkaufspreise, Produktionskosten und
              Lieferkosten sollten analysiert werden.
              Ziel ist eine nachhaltige Verbesserung der
              Stückmarge.
            </li>

            <li>
              <strong>Absatzstrategie prüfen:</strong>
              {#if deckungsbeitrag < 0}
                Mehr Verkäufe würden den Verlust aktuell
                weiter erhöhen. Deshalb sollte zuerst
                die Preis-Kosten-Struktur verbessert werden.
              {:else}
                Weitere Verkäufe würden die Fixkostendeckung
                nicht verbessern. Deshalb sollte zuerst
                ein positiver Deckungsbeitrag erzielt werden.
              {/if}
            </li>

            <li>
              <strong>Wirtschaftlichkeit neu bewerten:</strong>
              Vor weiteren Investitionen sollte eine neue
              Produktkalkulation durchgeführt werden.
            </li>
          </ol>


        {:else if gewinn < 0}

          <div class="status warnung">
            Optimierungsbedarf
          </div>

          <p>
            Das Unternehmen erzielt bereits einen positiven
            Deckungsbeitrag. Die geplante Absatzmenge reicht
            jedoch nicht aus, um die gesamten Fixkosten zu decken.
          </p>

          <h3>Empfohlene Maßnahmen</h3>

          <ol>
            <li>
              <strong>Absatz erhöhen:</strong>
              Für den Break-even fehlen aktuell
              {fehlendeMenge} Stück.
              Eine gezielte Steigerung des Absatzes
              sollte geprüft werden.
            </li>

            <li>
              <strong>Preisstrategie optimieren:</strong>
              {#if menge > 0}
                Bei unveränderter Absatzmenge wäre
                rechnerisch ein Verkaufspreis von mindestens
                {euro(notwendigerPreis)}
                erforderlich, um die Kosten zu decken.
              {:else}
                Zunächst sollte eine realistische
                Absatzplanung erstellt werden.
              {/if}
            </li>

            <li>
              <strong>Fixkosten kontrollieren:</strong>
              Laufende Kosten wie Personal, Miete,
              Verwaltung und Software sollten
              auf Einsparungsmöglichkeiten untersucht werden.
            </li>

            <li>
              <strong>Vertrieb verbessern:</strong>
              Marketing- und Vertriebsmaßnahmen sollten
              auf wirtschaftlich attraktive Zielgruppen
              ausgerichtet werden.
            </li>
          </ol>


        {:else if gewinn === 0}

          <div class="status warnung">
            Break-even erreicht
          </div>

          <p>
            Das Unternehmen deckt aktuell seine Kosten.
            Ein finanzieller Sicherheitspuffer ist jedoch
            noch nicht vorhanden.
          </p>

          <h3>Empfohlene Maßnahmen</h3>

          <ol>
            <li>
              <strong>Absatz steigern:</strong>
              Ziel sollte eine höhere Absatzmenge sein,
              um einen positiven Gewinn zu erzielen.
            </li>

            <li>
              <strong>Deckungsbeitrag erhöhen:</strong>
              Preisanpassungen und Kostensenkungen
              sollten geprüft werden.
            </li>

            <li>
              <strong>Risiken reduzieren:</strong>
              Ein Sicherheitspuffer für unerwartete
              Kosten oder Absatzschwankungen sollte
              aufgebaut werden.
            </li>
          </ol>


        {:else}

          <div class="status gut">
            Profitables Geschäftsmodell
          </div>

          <p>
            Die aktuelle Planung erzielt einen Gewinn von
            <strong>{euro(gewinn)}</strong>
            bei einer Gewinnmarge von
            <strong>{gewinnmarge.toFixed(2)} %</strong>.
          </p>

          <h3>Management-Empfehlung</h3>

          <ol>
            <li>
              <strong>Profitabilität sichern:</strong>
              Die bestehende Preis- und Kostenstruktur
              sollte regelmäßig überwacht werden.
            </li>

            <li>
              <strong>Sicherheitspuffer beobachten:</strong>
              Die geplante Absatzmenge könnte rechnerisch
              um rund {sicherheitspuffer.toFixed(1)} %
              sinken, bevor die Gewinnschwelle
              unterschritten wird.
            </li>

            <li>
              <strong>Profitables Wachstum prüfen:</strong>
              Zusätzliche Verkäufe können den Gewinn
              erhöhen, solange der Deckungsbeitrag
              stabil bleibt und keine zusätzlichen
              Fixkosten entstehen.
            </li>

            <li>
              <strong>Kostenentwicklung überwachen:</strong>
              Besonders steigende variable Kosten
              reduzieren unmittelbar den Deckungsbeitrag.
            </li>

            <li>
              <strong>Szenarioanalysen durchführen:</strong>
              Vor größeren Investitionen sollten
              Worst-Case-, Base-Case- und
              Best-Case-Szenarien betrachtet werden.
            </li>
          </ol>

        {/if}


        <div class="consulting-footer">
          Analysis provided by
          <strong>.Mittersteiner Consulting</strong>
        </div>

      </section>

    {/if}

  </div>
</main>


<style>
  :global(*) {
    box-sizing: border-box;
  }

  :global(:root) {
    font-family: Arial, sans-serif;
    color-scheme: light;
    color: #1e293b;
    background: #f4f6f8;
  }

  :global(body) {
    margin: 0;
    display: block;
    min-width: 320px;
    min-height: 100vh;
    background: #f4f6f8;
  }

  :global(#app) {
    width: 100%;
    max-width: none;
    margin: 0;
    padding: 0;
    text-align: left;
  }

  main {
    width: 100%;
    min-height: 100vh;
    padding: 30px 25px 50px;
  }

  .container {
    max-width: 1350px;
    margin: auto;
  }

  /* Firmenkopf */
  header {
    margin-bottom: 30px;
    text-align: center;
  }

  .firma {
    font-size: 25px;
    font-weight: bold;
    color: #111827;
  }

  .punkt {
    font-size: 35px;
    color: #1d4ed8;
  }

  .consulting {
    margin-left: 8px;
    font-weight: normal;
    color: #64748b;
  }

  .linie {
    width: 60px;
    height: 3px;
    margin: 12px auto 20px;
    background: #1d4ed8;
    border-radius: 5px;
  }

  .produkt-label {
    margin: 0 0 5px;
    font-size: 12px;
    font-weight: bold;
    letter-spacing: 2px;
    color: #64748b;
  }

  header h1 {
    margin: 0;
    font-size: 42px;
    color: #1d4ed8;
  }

  .untertitel {
    margin-top: 5px;
    font-size: 17px;
    color: #64748b;
  }

  /* Dashboard */
  .dashboard {
    display: grid;
    grid-template-columns: repeat(3, minmax(0, 1fr));
    gap: 20px;
  }

  .karte {
    min-width: 0;
    background: white;
    padding: 25px;
    border-radius: 14px;
    box-shadow: 0 3px 15px rgba(0, 0, 0, 0.07);
  }

  .karten-titel {
    margin-top: 0;
    margin-bottom: 25px;
    padding: 12px;
    border-radius: 8px;
    background: #dbeafe;
    color: #1d4ed8;
    text-align: center;
  }

  /* Eingabefelder */
  label {
    display: block;
    margin-bottom: 20px;
    font-weight: bold;
    text-align: center;
  }

  input {
    width: 100%;
    margin-top: 8px;
    padding: 12px;
    font-size: 16px;
    color: #172554;
    background: #eef4ff;
    border: 2px solid #c7d7f5;
    border-radius: 8px;
    transition: 0.2s;
  }

  input:hover {
    border-color: #93b4ed;
    background: #e7efff;
  }

  input:focus {
    outline: none;
    background: white;
    border-color: #2563eb;
    box-shadow: 0 0 0 3px #dbeafe;
  }

  .speicher-info {
    margin-top: 20px;
    font-size: 13px;
    color: #15803d;
    text-align: center;
  }

  .speicher-fehler {
    margin-top: 20px;
    font-size: 13px;
    color: #dc2626;
    text-align: center;
  }

  .fehler-karte {
    grid-column: span 2;
  }

  .fehler-karte h2 {
    color: #dc2626;
  }

  /* Ergebnisse */
  .ergebnis {
    display: flex;
    justify-content: space-between;
    gap: 15px;
    padding: 14px 0;
    border-bottom: 1px solid #e2e8f0;
  }

  .ergebnis:last-child {
    border-bottom: none;
  }

  .ergebnis strong {
    text-align: right;
  }

  /* Diagramm */
  .diagramm {
    margin-top: 10px;
  }

  .diagramm-zeile {
    margin-bottom: 28px;
  }

  .diagramm-info {
    display: flex;
    justify-content: space-between;
    gap: 10px;
    margin-bottom: 8px;
    flex-wrap: wrap;
  }

  .balken-hintergrund {
    width: 100%;
    height: 24px;
    background: #e5e7eb;
    border-radius: 6px;
    overflow: hidden;
  }

  .balken {
    height: 100%;
    border-radius: 6px;
    transition: width 0.4s ease;
  }

  .umsatz {
    background: #2563eb;
  }

  .kosten {
    background: #64748b;
  }

  .gewinn {
    background: #16a34a;
  }

  .verlust {
    background: #dc2626;
  }

  .diagramm-hinweis {
    margin-top: 30px;
    font-size: 14px;
    color: #64748b;
    text-align: center;
  }

  /* Ergebnisbox */
  .gewinn-box {
    margin-top: 20px;
    padding: 25px;
    background: white;
    border-radius: 14px;
    text-align: center;
    box-shadow: 0 3px 15px rgba(0, 0, 0, 0.07);
  }

  .kleine-ueberschrift {
    margin-top: 0;
    font-size: 12px;
    font-weight: bold;
    letter-spacing: 1.5px;
    color: #64748b;
  }

  .gewinn-box h2 {
    margin: 10px 0;
    font-size: 36px;
  }

  .positiv,
  .positiv-text {
    color: #15803d;
  }

  .negativ,
  .negativ-text {
    color: #dc2626;
  }

  .meldung {
    margin-bottom: 0;
    font-weight: bold;
    line-height: 1.6;
  }

  /* Unternehmensberatung */
  .beratung {
    margin-top: 20px;
    padding: 32px;
    background: white;
    border-radius: 14px;
    box-shadow: 0 3px 15px rgba(0, 0, 0, 0.07);
    text-align: left;
  }

  .beratung h2 {
    margin-top: 3px;
    margin-bottom: 20px;
    font-size: 27px;
    color: #1d4ed8;
    text-align: left;
  }

  .beratung h3 {
    margin-top: 25px;
    text-align: left;
  }

  .beratung p {
    line-height: 1.7;
    text-align: left;
  }

  .beratung ol {
    padding-left: 24px;
    margin-left: 0;
    text-align: left;
    list-style-position: outside;
  }

  .beratung li {
    margin-bottom: 15px;
    padding-left: 5px;
    line-height: 1.65;
    text-align: left;
  }

  .status {
    display: inline-block;
    margin-bottom: 15px;
    padding: 8px 13px;
    border-radius: 7px;
    font-weight: bold;
  }

  .gut {
    color: #166534;
    background: #dcfce7;
  }

  .warnung {
    color: #92400e;
    background: #fef3c7;
  }

  .kritisch {
    color: #991b1b;
    background: #fee2e2;
  }

  .consulting-footer {
    margin-top: 30px;
    padding-top: 20px;
    border-top: 1px solid #e2e8f0;
    color: #64748b;
    font-size: 14px;
    text-align: right;
  }

  .consulting-footer strong {
    color: #1e293b;
  }

  /* Tablet */
  @media (max-width: 1050px) {
    .dashboard {
      grid-template-columns: repeat(2, minmax(0, 1fr));
    }

    .dashboard .karte:last-child {
      grid-column: 1 / -1;
    }
  }

  /* Smartphone */
  @media (max-width: 700px) {
    main {
      padding: 20px 12px 40px;
    }

    .dashboard {
      grid-template-columns: 1fr;
    }

    .dashboard .karte:last-child,
    .fehler-karte {
      grid-column: auto;
    }

    header h1 {
      font-size: 34px;
    }

    .firma {
      font-size: 21px;
    }

    .beratung {
      padding: 22px;
    }
  }
</style>
