<script lang="ts">
  // Eingabewerte
  let verkaufspreis = 50;
  let variableKosten = 30;
  let fixkosten = 20000;
  let menge = 1500;

  // Berechnungen
  $: umsatz = verkaufspreis * menge;

  $: deckungsbeitrag = verkaufspreis - variableKosten;

  $: gesamtkosten = fixkosten + variableKosten * menge;

  $: gewinn = umsatz - gesamtkosten;

  $: breakEven =
    deckungsbeitrag > 0
      ? fixkosten / deckungsbeitrag
      : 0;

  $: gewinnmarge =
    umsatz > 0
      ? (gewinn / umsatz) * 100
      : 0;


  // Diagramm
  $: groessterWert = Math.max(
    umsatz,
    gesamtkosten,
    Math.abs(gewinn),
    1
  );

  $: umsatzBreite = (umsatz / groessterWert) * 100;
  $: kostenBreite = (gesamtkosten / groessterWert) * 100;
  $: gewinnBreite = (Math.abs(gewinn) / groessterWert) * 100;


  // Werte für Handlungsempfehlungen
  $: fehlendeMenge =
    deckungsbeitrag > 0
      ? Math.max(0, Math.ceil(breakEven - menge))
      : 0;

  $: sicherheitspuffer =
    deckungsbeitrag > 0 && menge > 0
      ? ((menge - breakEven) / menge) * 100
      : 0;

  $: notwendigerPreis =
    menge > 0
      ? variableKosten + fixkosten / menge
      : 0;


  // Kurze Analyse
  $: erklaerung = "";

  $: if (deckungsbeitrag <= 0) {
    erklaerung =
      "Mit der aktuellen Preis- und Kostenstruktur kann kein Break-even erreicht werden.";
  } else if (gewinn < 0) {
    erklaerung =
      `Die erwartete Absatzmenge liegt unter dem Break-even-Point von ${Math.ceil(
        breakEven
      )} Stück.`;
  } else if (gewinn === 0) {
    erklaerung =
      "Der Break-even-Point wurde genau erreicht. Alle Kosten werden gedeckt.";
  } else {
    erklaerung =
      `Die erwartete Absatzmenge liegt über dem Break-even-Point von ${Math.ceil(
        breakEven
      )} Stück. Das Unternehmen arbeitet profitabel.`;
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

      <br>

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
            bind:value={verkaufspreis}
          />
        </label>

        <label>
          Variable Kosten pro Stück (€)

          <input
            type="number"
            min="0"
            bind:value={variableKosten}
          />
        </label>

        <label>
          Fixkosten (€)

          <input
            type="number"
            min="0"
            bind:value={fixkosten}
          />
        </label>

        <label>
          Erwartete Absatzmenge

          <input
            type="number"
            min="0"
            bind:value={menge}
          />
        </label>

      </section>


      <!-- Ergebnisse -->
      <section class="karte">

        <h2 class="karten-titel">
          Ergebnisse
        </h2>

        <div class="ergebnis">
          <span>Umsatz</span>
          <strong>€ {umsatz.toFixed(2)}</strong>
        </div>

        <div class="ergebnis">
          <span>Gesamtkosten</span>
          <strong>€ {gesamtkosten.toFixed(2)}</strong>
        </div>

        <div class="ergebnis">
          <span>Deckungsbeitrag / Stück</span>
          <strong>€ {deckungsbeitrag.toFixed(2)}</strong>
        </div>

        <div class="ergebnis">
          <span>Break-even-Point</span>

          {#if deckungsbeitrag > 0}
            <strong>{Math.ceil(breakEven)} Stück</strong>
          {:else}
            <strong>Nicht möglich</strong>
          {/if}
        </div>

        <div class="ergebnis">
          <span>Gewinnmarge</span>
          <strong>{gewinnmarge.toFixed(2)} %</strong>
        </div>

      </section>


      <!-- Diagramm -->
      <section class="karte">

        <h2 class="karten-titel">
          Financial Overview
        </h2>

        <div class="diagramm">

          <!-- Umsatz -->
          <div class="diagramm-zeile">

            <div class="diagramm-info">
              <span>Umsatz</span>
              <strong>€ {umsatz.toFixed(0)}</strong>
            </div>

            <div class="balken-hintergrund">
              <div
                class="balken umsatz"
                style="width: {umsatzBreite}%"
              ></div>
            </div>

          </div>


          <!-- Kosten -->
          <div class="diagramm-zeile">

            <div class="diagramm-info">
              <span>Gesamtkosten</span>
              <strong>€ {gesamtkosten.toFixed(0)}</strong>
            </div>

            <div class="balken-hintergrund">
              <div
                class="balken kosten"
                style="width: {kostenBreite}%"
              ></div>
            </div>

          </div>


          <!-- Gewinn oder Verlust -->
          <div class="diagramm-zeile">

            <div class="diagramm-info">

              {#if gewinn >= 0}
                <span>Gewinn</span>
              {:else}
                <span>Verlust</span>
              {/if}

              <strong>
                € {Math.abs(gewinn).toFixed(0)}
              </strong>

            </div>

            <div class="balken-hintergrund">
              <div
                class="balken"
                class:gewinn={gewinn >= 0}
                class:verlust={gewinn < 0}
                style="width: {gewinnBreite}%"
              ></div>
            </div>

          </div>

        </div>

        <p class="diagramm-hinweis">
          Live-Analyse basierend auf den Unternehmensdaten
        </p>

      </section>

    </div>


    <!-- Ergebnis -->
    <section class="gewinn-box">

      <p class="kleine-ueberschrift">
        ERWARTETES ERGEBNIS
      </p>

      {#if gewinn > 0}

        <h2 class="positiv">
          + € {gewinn.toFixed(2)}
        </h2>

      {:else if gewinn < 0}

        <h2 class="negativ">
          - € {Math.abs(gewinn).toFixed(2)}
        </h2>

      {:else}

        <h2>
          € 0.00
        </h2>

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

      <h2>
        Handlungsempfehlung
      </h2>


      {#if deckungsbeitrag <= 0}

        <div class="status kritisch">
          Kritischer Handlungsbedarf
        </div>

        <p>
          Die aktuelle Preis- und Kostenstruktur ist wirtschaftlich
          nicht tragfähig. Mit jedem zusätzlich verkauften Stück
          verschlechtert sich das Ergebnis.
        </p>

        <h3>Empfohlene Maßnahmen</h3>

        <ol>

          <li>
            <strong>Preisstrategie überarbeiten:</strong>
            Der Verkaufspreis von € {verkaufspreis.toFixed(2)}
            reicht nicht aus, um die variablen Kosten von
            € {variableKosten.toFixed(2)} zu decken.
          </li>

          <li>
            <strong>Variable Kosten reduzieren:</strong>
            Einkaufs-, Produktions- und Lieferkosten sollten
            detailliert analysiert werden.
          </li>

          <li>
            <strong>Skalierung vermeiden:</strong>
            Eine höhere Absatzmenge würde den Verlust aktuell
            weiter erhöhen.
          </li>

          <li>
            <strong>Produktkalkulation neu bewerten:</strong>
            Ziel sollte zuerst ein positiver Deckungsbeitrag
            pro verkauftem Stück sein.
          </li>

        </ol>


      {:else if gewinn < 0}

        <div class="status warnung">
          Optimierungsbedarf
        </div>

        <p>
          Das Produkt erzielt einen positiven Deckungsbeitrag.
          Die geplante Absatzmenge reicht jedoch noch nicht aus,
          um die Fixkosten vollständig zu decken.
        </p>

        <h3>Empfohlene Maßnahmen</h3>

        <ol>

          <li>
            <strong>Absatz steigern:</strong>
            Für den Break-even fehlen aktuell ungefähr
            {fehlendeMenge} Stück.
          </li>

          <li>
            <strong>Preisstrategie prüfen:</strong>
            Bei unveränderter Absatzmenge müsste der Verkaufspreis
            ungefähr € {notwendigerPreis.toFixed(2)}
            betragen, um den Break-even zu erreichen.
          </li>

          <li>
            <strong>Fixkosten analysieren:</strong>
            Bereiche wie Personal, Miete, Verwaltung oder Software
            sollten auf Einsparungsmöglichkeiten untersucht werden.
          </li>

          <li>
            <strong>Vertrieb verbessern:</strong>
            Marketing und Vertrieb sollten gezielt dort verstärkt
            werden, wo zusätzliche Verkäufe mit möglichst geringen
            Zusatzkosten erreichbar sind.
          </li>

        </ol>


      {:else if gewinn === 0}

        <div class="status warnung">
          Break-even erreicht
        </div>

        <p>
          Das Unternehmen deckt aktuell genau seine Kosten.
          Ein ausreichender finanzieller Sicherheitspuffer besteht
          allerdings noch nicht.
        </p>

        <h3>Empfohlene Maßnahmen</h3>

        <ol>

          <li>
            Absatzmenge über den aktuellen Break-even-Point erhöhen.
          </li>

          <li>
            Deckungsbeitrag durch höhere Preise oder niedrigere
            variable Kosten verbessern.
          </li>

          <li>
            Einen finanziellen Sicherheitspuffer für unerwartete
            Kosten oder Absatzschwankungen aufbauen.
          </li>

        </ol>


      {:else}

        <div class="status gut">
          Profitables Geschäftsmodell
        </div>

        <p>
          Die aktuelle Planung erzielt einen Gewinn von
          <strong>€ {gewinn.toFixed(2)}</strong>
          bei einer Gewinnmarge von
          <strong>{gewinnmarge.toFixed(2)} %</strong>.
        </p>

        <h3>Management-Empfehlung</h3>

        <ol>

          <li>
            <strong>Profitabilität sichern:</strong>
            Die bestehende Preis- und Kostenstruktur sollte
            regelmäßig überwacht werden.
          </li>

          <li>
            <strong>Sicherheitspuffer beobachten:</strong>
            Die geplante Absatzmenge liegt aktuell rund
            {Math.max(0, sicherheitspuffer).toFixed(1)} %
            über der Break-even-Menge.
          </li>

          <li>
            <strong>Profitables Wachstum prüfen:</strong>
            Zusätzliche Verkäufe können den Gewinn erhöhen,
            solange der Deckungsbeitrag stabil bleibt.
          </li>

          <li>
            <strong>Kostenentwicklung überwachen:</strong>
            Besonders steigende variable Kosten wirken sich
            direkt negativ auf den Deckungsbeitrag aus.
          </li>

          <li>
            <strong>Szenarioanalysen durchführen:</strong>
            Vor größeren Entscheidungen sollten zusätzlich
            Worst-Case-, Base-Case- und Best-Case-Szenarien
            betrachtet werden.
          </li>

        </ol>

      {/if}


      <div class="consulting-footer">
        Analysis provided by
        <strong>.Mittersteiner Consulting</strong>
      </div>

    </section>

  </div>

</main>


<style>

  :global(*) {
    box-sizing: border-box;
  }


  :global(body) {
    margin: 0;

    min-width: 320px;
    min-height: 100vh;

    background: #f4f6f8;

    font-family: Arial, sans-serif;

    color: #1e293b;
  }


  :global(#app) {
    width: 100%;

    max-width: none;

    margin: 0;
    padding: 0;
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


  /* Header */
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

    grid-template-columns: 1fr 1fr 1fr;

    gap: 20px;
  }


  .karte {
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

    margin-bottom: 8px;
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


  .positiv {
    color: #15803d;
  }


  .negativ {
    color: #dc2626;
  }


  .positiv-text {
    color: #15803d;
  }


  .negativ-text {
    color: #dc2626;
  }


  .meldung {
    margin-bottom: 0;

    font-weight: bold;

    line-height: 1.6;
  }


  /* Beratung */
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
      grid-template-columns: 1fr 1fr;
    }

    .dashboard .karte:last-child {
      grid-column: 1 / 3;
    }

  }


  /* Handy */
  @media (max-width: 700px) {

    main {
      padding: 20px 12px 40px;
    }

    .dashboard {
      grid-template-columns: 1fr;
    }

    .dashboard .karte:last-child {
      grid-column: auto;
    }

    header h1 {
      font-size: 34px;
    }

    .firma {
      font-size: 21px;
    }

  }

</style>