<script lang="ts">
	// === INPUTS ===
	let aankoopprijs = $state(250000);
	let eigenGeld = $state(30000);
	let hypotheekrente = $state(3.4);
	let looptijdJaren = $state(25);
	let verblijfsduurJaren = $state(3);
	let waardestijging = $state(3);
	let maandelijksehuur = $state(875);
	let gwePerMaand = $state(165);
	let huurServicekosten = $state(125);
	let garagebox_huur = $state(115);
	let brutoMaandinkomen = $state(4950);
	let nettoMaandinkomen = $state(3500);
	let vmeBijdrage = $state(200);
	let sbiBijdrage = $state(70);

	// === DERIVED CALCULATIONS ===
	let notarisAankoopakte = $derived(Math.min(4000, Math.max(2000, aankoopprijs * 0.01)));
	let notarisKredietakte = 2500;
	let bankkosten = 800;
	let registratierechten = $derived(aankoopprijs * 0.02);
	let bijkomendeKosten = $derived(registratierechten + notarisAankoopakte + notarisKredietakte + bankkosten);
	let eigenInbrengOpPrijs = $derived(eigenGeld - bijkomendeKosten);
	let hypotheekbedrag = $derived(aankoopprijs - Math.max(0, eigenInbrengOpPrijs));
	let quotiteit = $derived(hypotheekbedrag / aankoopprijs);

	let maandRente = $derived(hypotheekrente / 100 / 12);
	let aantalMaanden = $derived(looptijdJaren * 12);
	let maandlastHypotheek = $derived(
		maandRente > 0
			? hypotheekbedrag * (maandRente * Math.pow(1 + maandRente, aantalMaanden)) / (Math.pow(1 + maandRente, aantalMaanden) - 1)
			: hypotheekbedrag / aantalMaanden
	);

	// Eigenwoningforfait
	let eigenwoningforfait = $derived(aankoopprijs * 0.0035);
	let marginaalTarief = $derived(brutoMaandinkomen * 12 > 75518 ? 0.4950 : 0.3697);

	// HRA berekening per jaar
	function berekenRenteJaar(jaar: number): number {
		const r = maandRente;
		const n = aantalMaanden;
		let renteJaar = 0;
		for (let m = (jaar - 1) * 12; m < jaar * 12; m++) {
			const restschuld = hypotheekbedrag * (Math.pow(1 + r, n) - Math.pow(1 + r, m)) / (Math.pow(1 + r, n) - 1);
			renteJaar += restschuld * r;
		}
		return renteJaar;
	}

	let hraPerJaar = $derived(
		Array.from({ length: verblijfsduurJaren }, (_, i) => {
			const rente = berekenRenteJaar(i + 1);
			const nettoAftrek = rente - eigenwoningforfait;
			const teruggave = Math.max(0, nettoAftrek) * Math.min(marginaalTarief, 0.3697);
			return { jaar: i + 1, rente, eigenwoningforfait, nettoAftrek, teruggave };
		})
	);

	let totaalHRA = $derived(hraPerJaar.reduce((s, j) => s + j.teruggave, 0));
	let maandelijkseHRA = $derived(totaalHRA / (verblijfsduurJaren * 12));

	// Restschuld na verblijfsduur
	let betaaldeMaanden = $derived(verblijfsduurJaren * 12);
	let restschuld = $derived(
		hypotheekbedrag * (Math.pow(1 + maandRente, aantalMaanden) - Math.pow(1 + maandRente, betaaldeMaanden)) /
		(Math.pow(1 + maandRente, aantalMaanden) - 1)
	);

	// Woningwaarde na verblijfsduur
	let woningwaardeNa = $derived(aankoopprijs * Math.pow(1 + waardestijging / 100, verblijfsduurJaren));
	let nettoVerkoopopbrengst = $derived(woningwaardeNa * 0.97);
	let nettoTerugontvangen = $derived(nettoVerkoopopbrengst - restschuld);

	// Maandlasten kopen
	let brandverzekering = 25;
	let onroerendeVoorheffingMaand = 70;
	let onderhoudsreserve = 40;
	let maandlastKopenBruto = $derived(
		maandlastHypotheek + gwePerMaand + vmeBijdrage + sbiBijdrage + brandverzekering + onroerendeVoorheffingMaand + onderhoudsreserve
	);
	let maandlastKopenNetto = $derived(maandlastKopenBruto - maandelijkseHRA);

	// Maandlasten huren
	let maandlastHuren = $derived(maandelijksehuur + gwePerMaand + huurServicekosten + garagebox_huur);

	// Totale kosten over verblijfsduur
	let totaalKopen = $derived(maandlastKopenNetto * 12 * verblijfsduurJaren + bijkomendeKosten);
	let nettoKostenKopen = $derived(totaalKopen - nettoTerugontvangen);
	let opportuniteitskosten = $derived(eigenGeld * 0.04 * verblijfsduurJaren);
	let nettoKostenHuren = $derived(maandlastHuren * 12 * verblijfsduurJaren + opportuniteitskosten);
	let voordeelKopen = $derived(nettoKostenHuren - nettoKostenKopen);

	// Breakeven
	function berekenBreakeven(): number {
		for (let pct = -10; pct <= 15; pct += 0.1) {
			const ww = aankoopprijs * Math.pow(1 + pct / 100, verblijfsduurJaren);
			const nvo = ww * 0.97;
			const nto = nvo - restschuld;
			const nkk = totaalKopen - nto;
			if (nkk <= nettoKostenHuren) return pct;
		}
		return -10;
	}
	let breakeven = $derived(berekenBreakeven());

	// Scenario's voor waardestijging chart
	let scenarios = $derived(
		[-3, -1, 0, 1, 3, 5, 7].map(pct => {
			const ww = aankoopprijs * Math.pow(1 + pct / 100, verblijfsduurJaren);
			const nvo = ww * 0.97;
			const nto = nvo - restschuld;
			const nkk = totaalKopen - nto;
			return { pct, nkk, nkh: nettoKostenHuren, voordeel: nettoKostenHuren - nkk };
		})
	);

	// Warnings
	let warnings = $derived((() => {
		const w: { type: string; msg: string }[] = [];
		if (eigenInbrengOpPrijs < 0) w.push({ type: 'error', msg: 'Eigen geld onvoldoende voor bijkomende kosten!' });
		if (quotiteit > 0.92) w.push({ type: 'error', msg: `Quotiteit ${fmt(quotiteit * 100, 1)}% — financiering mogelijk moeilijk` });
		else if (quotiteit > 0.90) w.push({ type: 'warn', msg: `Quotiteit ${fmt(quotiteit * 100, 1)}% — verwacht renteopslag 0,10–0,25%` });
		if (verblijfsduurJaren < 3) w.push({ type: 'warn', msg: 'Korter dan 3 jaar verhoogt naheffingsrisico registratierechten' });
		return w;
	})());

	// Quotiteit bar data
	let quotiteitKleur = $derived(quotiteit <= 0.80 ? '#22c55e' : quotiteit <= 0.90 ? '#f59e0b' : '#ef4444');

	function fmt(n: number, decimals = 0): string {
		return n.toLocaleString('nl-NL', { minimumFractionDigits: decimals, maximumFractionDigits: decimals });
	}

	function eur(n: number): string {
		return '€' + fmt(n);
	}

	// Chart bars max
	let chartMax = $derived(Math.max(...scenarios.map(s => Math.max(Math.abs(s.nkk), Math.abs(s.nkh)))));
</script>

<svelte:head>
	<title>Woning Kopen in België — Calculator</title>
	<link rel="preconnect" href="https://fonts.googleapis.com">
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous">
	<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
</svelte:head>

<div class="page">
	<!-- HEADER -->
	<header>
		<div class="header-inner">
			<div class="logo">🏠</div>
			<div>
				<h1>Woning Kopen in België</h1>
				<p class="subtitle">Financiële calculator voor Nederlanders — kopen vs. huren in Vlaanderen</p>
			</div>
		</div>
	</header>

	<!-- WARNINGS -->
	{#if warnings.length > 0}
		<div class="warnings">
			{#each warnings as w}
				<div class="warning {w.type}">
					<span class="warning-icon">{w.type === 'error' ? '⚠' : '⚡'}</span>
					{w.msg}
				</div>
			{/each}
		</div>
	{/if}

	<div class="grid-main">
		<!-- LEFT: INPUTS -->
		<section class="card inputs-card">
			<h2>Parameters</h2>

			<div class="input-group">
				<label>
					<span class="label-row"><span>Aankoopprijs</span><span class="val">{eur(aankoopprijs)}</span></span>
					<input type="range" min="150000" max="500000" step="5000" bind:value={aankoopprijs}>
				</label>
				<label>
					<span class="label-row"><span>Eigen geld</span><span class="val">{eur(eigenGeld)}</span></span>
					<input type="range" min="0" max="200000" step="1000" bind:value={eigenGeld}>
				</label>
				<label>
					<span class="label-row"><span>Hypotheekrente</span><span class="val">{fmt(hypotheekrente, 1)}%</span></span>
					<input type="range" min="1" max="6" step="0.1" bind:value={hypotheekrente}>
				</label>
				<label>
					<span class="label-row"><span>Looptijd</span><span class="val">{looptijdJaren} jaar</span></span>
					<input type="range" min="10" max="30" step="1" bind:value={looptijdJaren}>
				</label>
				<label>
					<span class="label-row"><span>Verblijfsduur</span><span class="val">{verblijfsduurJaren} jaar</span></span>
					<input type="range" min="1" max="10" step="1" bind:value={verblijfsduurJaren}>
				</label>
				<label>
					<span class="label-row"><span>Verwachte waardestijging</span><span class="val">{fmt(waardestijging, 1)}%/jr</span></span>
					<input type="range" min="-5" max="10" step="0.5" bind:value={waardestijging}>
				</label>
			</div>

			<h3>Huurkosten</h3>
			<div class="input-group">
				<label>
					<span class="label-row"><span>Kale huur</span><span class="val">{eur(maandelijksehuur)}</span></span>
					<input type="range" min="500" max="2500" step="25" bind:value={maandelijksehuur}>
				</label>
				<label>
					<span class="label-row"><span>Servicekosten huur</span><span class="val">{eur(huurServicekosten)}</span></span>
					<input type="range" min="0" max="300" step="5" bind:value={huurServicekosten}>
				</label>
				<label>
					<span class="label-row"><span>Garagebox huur</span><span class="val">{eur(garagebox_huur)}</span></span>
					<input type="range" min="0" max="200" step="5" bind:value={garagebox_huur}>
				</label>
			</div>

			<h3>Koopkosten</h3>
			<div class="input-group">
				<label>
					<span class="label-row"><span>GWE (gas/water/elektra)</span><span class="val">{eur(gwePerMaand)}</span></span>
					<input type="range" min="80" max="300" step="5" bind:value={gwePerMaand}>
				</label>
				<label>
					<span class="label-row"><span>VME-bijdrage (syndic)</span><span class="val">{eur(vmeBijdrage)}</span></span>
					<input type="range" min="50" max="350" step="10" bind:value={vmeBijdrage}>
				</label>
				<label>
					<span class="label-row"><span>Schuldsaldoverzekering</span><span class="val">{eur(sbiBijdrage)}</span></span>
					<input type="range" min="30" max="120" step="5" bind:value={sbiBijdrage}>
				</label>
			</div>

			<h3>Inkomen</h3>
			<div class="input-group">
				<label>
					<span class="label-row"><span>Bruto maandinkomen</span><span class="val">{eur(brutoMaandinkomen)}</span></span>
					<input type="range" min="2000" max="15000" step="50" bind:value={brutoMaandinkomen}>
				</label>
				<label>
					<span class="label-row"><span>Netto maandinkomen</span><span class="val">{eur(nettoMaandinkomen)}</span></span>
					<input type="range" min="1500" max="10000" step="50" bind:value={nettoMaandinkomen}>
				</label>
			</div>
		</section>

		<!-- RIGHT: RESULTS -->
		<div class="results-col">

			<!-- VERDICT -->
			<section class="card verdict-card" class:positive={voordeelKopen > 0} class:negative={voordeelKopen <= 0}>
				<div class="verdict-header">
					<span class="verdict-label">{voordeelKopen > 0 ? 'Kopen is voordeliger' : 'Huren is voordeliger'}</span>
					<span class="verdict-amount">{eur(Math.abs(voordeelKopen))}</span>
				</div>
				<p class="verdict-sub">over {verblijfsduurJaren} jaar bij {fmt(waardestijging, 1)}% waardestijging/jaar</p>
				<div class="verdict-breakeven">Breakeven waardestijging: <strong>{fmt(breakeven, 1)}%/jaar</strong></div>
			</section>

			<!-- QUOTITEIT -->
			<section class="card compact">
				<div class="row-between">
					<span class="metric-label">Quotiteit</span>
					<span class="metric-value" style="color: {quotiteitKleur}">{fmt(quotiteit * 100, 1)}%</span>
				</div>
				<div class="bar-track">
					<div class="bar-fill" style="width: {Math.min(quotiteit * 100, 100)}%; background: {quotiteitKleur}"></div>
					<div class="bar-markers">
						<div class="bar-marker" style="left: 80%"><span>80%</span></div>
						<div class="bar-marker" style="left: 90%"><span>90%</span></div>
					</div>
				</div>
				<div class="row-between sub-metrics">
					<span>Hypotheek: {eur(hypotheekbedrag)}</span>
					<span>Eigen inbreng: {eur(Math.max(0, eigenInbrengOpPrijs))}</span>
				</div>
			</section>

			<!-- MONTHLY COMPARISON -->
			<section class="card compact">
				<h2>Maandlasten vergelijking</h2>
				<div class="month-compare">
					<div class="month-col">
						<div class="month-title">Kopen</div>
						<div class="month-amount">{eur(maandlastKopenNetto)}<span class="month-small">/mnd</span></div>
						<div class="month-bar-stack">
							<div class="bar-seg" style="flex: {maandlastHypotheek}; background: var(--green-600)" title="Hypotheek {eur(maandlastHypotheek)}"></div>
							<div class="bar-seg" style="flex: {gwePerMaand}; background: var(--green-400)" title="GWE {eur(gwePerMaand)}"></div>
							<div class="bar-seg" style="flex: {vmeBijdrage}; background: var(--green-300)" title="VME {eur(vmeBijdrage)}"></div>
							<div class="bar-seg" style="flex: {sbiBijdrage + brandverzekering + onroerendeVoorheffingMaand + onderhoudsreserve}; background: var(--green-200)" title="Overig {eur(sbiBijdrage + brandverzekering + onroerendeVoorheffingMaand + onderhoudsreserve)}"></div>
						</div>
						<div class="month-details">
							<div class="detail-row"><span>Hypotheek</span><span>{eur(maandlastHypotheek)}</span></div>
							<div class="detail-row"><span>GWE</span><span>{eur(gwePerMaand)}</span></div>
							<div class="detail-row"><span>VME / syndic</span><span>{eur(vmeBijdrage)}</span></div>
							<div class="detail-row"><span>SBI</span><span>{eur(sbiBijdrage)}</span></div>
							<div class="detail-row"><span>Brand + OVV + onderhoud</span><span>{eur(brandverzekering + onroerendeVoorheffingMaand + onderhoudsreserve)}</span></div>
							<div class="detail-row hra"><span>HRA teruggave</span><span>-{eur(maandelijkseHRA)}</span></div>
						</div>
					</div>
					<div class="month-col">
						<div class="month-title">Huren</div>
						<div class="month-amount">{eur(maandlastHuren)}<span class="month-small">/mnd</span></div>
						<div class="month-bar-stack">
							<div class="bar-seg" style="flex: {maandelijksehuur}; background: var(--slate-500)" title="Huur {eur(maandelijksehuur)}"></div>
							<div class="bar-seg" style="flex: {gwePerMaand}; background: var(--slate-400)" title="GWE {eur(gwePerMaand)}"></div>
							<div class="bar-seg" style="flex: {huurServicekosten}; background: var(--slate-300)" title="Service {eur(huurServicekosten)}"></div>
							<div class="bar-seg" style="flex: {garagebox_huur}; background: var(--slate-200)" title="Garage {eur(garagebox_huur)}"></div>
						</div>
						<div class="month-details">
							<div class="detail-row"><span>Kale huur</span><span>{eur(maandelijksehuur)}</span></div>
							<div class="detail-row"><span>GWE</span><span>{eur(gwePerMaand)}</span></div>
							<div class="detail-row"><span>Servicekosten</span><span>{eur(huurServicekosten)}</span></div>
							<div class="detail-row"><span>Garagebox</span><span>{eur(garagebox_huur)}</span></div>
						</div>
					</div>
				</div>
			</section>

			<!-- COST WATERFALL -->
			<section class="card compact">
				<h2>Netto kostenvergelijking over {verblijfsduurJaren} jaar</h2>
				<div class="waterfall">
					<div class="wf-row">
						<span class="wf-label">Totaal uitgaven kopen</span>
						<div class="wf-bar-wrap">
							<div class="wf-bar neg" style="width: {Math.min(totaalKopen / Math.max(totaalKopen, nettoKostenHuren) * 100, 100)}%">{eur(totaalKopen)}</div>
						</div>
					</div>
					<div class="wf-row">
						<span class="wf-label">Terugontvangen verkoop</span>
						<div class="wf-bar-wrap">
							<div class="wf-bar pos" style="width: {Math.min(Math.abs(nettoTerugontvangen) / Math.max(totaalKopen, nettoKostenHuren) * 100, 100)}%">{eur(nettoTerugontvangen)}</div>
						</div>
					</div>
					<div class="wf-row highlight">
						<span class="wf-label"><strong>Netto kopen</strong></span>
						<div class="wf-bar-wrap">
							<div class="wf-bar {nettoKostenKopen > 0 ? 'neg' : 'pos'}" style="width: {Math.min(Math.abs(nettoKostenKopen) / Math.max(totaalKopen, nettoKostenHuren) * 100, 100)}%"><strong>{eur(nettoKostenKopen)}</strong></div>
						</div>
					</div>
					<div class="wf-divider"></div>
					<div class="wf-row">
						<span class="wf-label">Totaal uitgaven huren</span>
						<div class="wf-bar-wrap">
							<div class="wf-bar neutral" style="width: {Math.min(maandlastHuren * 12 * verblijfsduurJaren / Math.max(totaalKopen, nettoKostenHuren) * 100, 100)}%">{eur(maandlastHuren * 12 * verblijfsduurJaren)}</div>
						</div>
					</div>
					<div class="wf-row">
						<span class="wf-label">Opportuniteitskosten spaargeld</span>
						<div class="wf-bar-wrap">
							<div class="wf-bar neutral-light" style="width: {Math.min(opportuniteitskosten / Math.max(totaalKopen, nettoKostenHuren) * 100, 100)}%">{eur(opportuniteitskosten)}</div>
						</div>
					</div>
					<div class="wf-row highlight">
						<span class="wf-label"><strong>Netto huren</strong></span>
						<div class="wf-bar-wrap">
							<div class="wf-bar neutral" style="width: {Math.min(nettoKostenHuren / Math.max(totaalKopen, nettoKostenHuren) * 100, 100)}%"><strong>{eur(nettoKostenHuren)}</strong></div>
						</div>
					</div>
				</div>
			</section>

			<!-- SCENARIO TABLE -->
			<section class="card compact">
				<h2>Scenario's waardestijging</h2>
				<div class="scenario-chart">
					{#each scenarios as s}
						<div class="sc-row">
							<span class="sc-label">{s.pct > 0 ? '+' : ''}{s.pct}%</span>
							<div class="sc-bars">
								<div class="sc-bar kopen" style="width: {Math.abs(s.nkk) / chartMax * 45}%" class:profit={s.nkk < 0}>
									{eur(s.nkk)}
								</div>
								<div class="sc-bar huren" style="width: {s.nkh / chartMax * 45}%">
									{eur(s.nkh)}
								</div>
							</div>
							<span class="sc-verdict" class:positive={s.voordeel > 0} class:negative={s.voordeel <= 0}>
								{s.voordeel > 0 ? '+' : ''}{eur(s.voordeel)}
							</span>
						</div>
					{/each}
					<div class="sc-legend">
						<span><span class="dot kopen"></span> Netto kosten kopen</span>
						<span><span class="dot huren"></span> Netto kosten huren</span>
					</div>
				</div>
			</section>

			<!-- HRA TABLE -->
			<section class="card compact">
				<h2>Hypotheekrenteaftrek (HRA)</h2>
				<table>
					<thead>
						<tr><th>Jaar</th><th>Betaalde rente</th><th>Eigenwoning­forfait</th><th>Netto aftrek</th><th>Teruggave ({fmt(Math.min(marginaalTarief, 0.3697) * 100, 2)}%)</th></tr>
					</thead>
					<tbody>
						{#each hraPerJaar as j}
							<tr>
								<td>{j.jaar}</td>
								<td>{eur(j.rente)}</td>
								<td>{eur(j.eigenwoningforfait)}</td>
								<td>{eur(j.nettoAftrek)}</td>
								<td class="hra-cell">{eur(j.teruggave)}</td>
							</tr>
						{/each}
						<tr class="total-row">
							<td>Totaal</td>
							<td>{eur(hraPerJaar.reduce((s, j) => s + j.rente, 0))}</td>
							<td>{eur(hraPerJaar.reduce((s, j) => s + j.eigenwoningforfait, 0))}</td>
							<td>{eur(hraPerJaar.reduce((s, j) => s + j.nettoAftrek, 0))}</td>
							<td class="hra-cell">{eur(totaalHRA)}</td>
						</tr>
					</tbody>
				</table>
				<p class="table-note">≈ {eur(maandelijkseHRA)}/maand via voorlopige teruggave</p>
			</section>

			<!-- EENMALIGE KOSTEN -->
			<section class="card compact">
				<h2>Eenmalige aankoopkosten</h2>
				<table>
					<thead><tr><th>Post</th><th>Bedrag</th></tr></thead>
					<tbody>
						<tr><td>Registratierechten (2%)</td><td>{eur(registratierechten)}</td></tr>
						<tr><td>Notaris aankoopakte</td><td>{eur(notarisAankoopakte)}</td></tr>
						<tr><td>Notaris kredietakte</td><td>{eur(notarisKredietakte)}</td></tr>
						<tr><td>Bankdossier + schatting</td><td>{eur(bankkosten)}</td></tr>
						<tr class="total-row"><td><strong>Totaal bijkomende kosten</strong></td><td><strong>{eur(bijkomendeKosten)}</strong></td></tr>
					</tbody>
				</table>
				<p class="table-note">Besparing t.o.v. 12% standaardtarief: {eur(aankoopprijs * 0.10)}</p>
			</section>

			<!-- BETAALBAARHEID -->
			<section class="card compact">
				<h2>Betaalbaarheid</h2>
				<div class="afford-grid">
					<div class="afford-item">
						<span class="afford-label">Hypotheek % netto inkomen</span>
						<span class="afford-value" style="color: {maandlastHypotheek / nettoMaandinkomen > 0.35 ? '#ef4444' : '#22c55e'}">{fmt(maandlastHypotheek / nettoMaandinkomen * 100, 1)}%</span>
						<span class="afford-note">Norm: max 33–35%</span>
					</div>
					<div class="afford-item">
						<span class="afford-label">Vrij besteedbaar na kopen</span>
						<span class="afford-value">{eur(nettoMaandinkomen - maandlastKopenNetto)}</span>
						<span class="afford-note">per maand</span>
					</div>
					<div class="afford-item">
						<span class="afford-label">Vrij besteedbaar na huren</span>
						<span class="afford-value">{eur(nettoMaandinkomen - maandlastHuren)}</span>
						<span class="afford-note">per maand</span>
					</div>
				</div>
			</section>
		</div>
	</div>

	<footer>
		<p>Indicatieve berekening — raadpleeg een belastingadviseur voor persoonlijk advies. Registratierechten 2% geldt bij enige eigen woning in Vlaanderen. Stand maart 2026.</p>
	</footer>
</div>

<style>
	:root {
		--green-50: #f0fdf4;
		--green-100: #dcfce7;
		--green-200: #bbf7d0;
		--green-300: #86efac;
		--green-400: #4ade80;
		--green-500: #22c55e;
		--green-600: #16a34a;
		--green-700: #15803d;
		--green-800: #166534;
		--green-900: #14532d;
		--slate-50: #f8fafc;
		--slate-100: #f1f5f9;
		--slate-200: #e2e8f0;
		--slate-300: #cbd5e1;
		--slate-400: #94a3b8;
		--slate-500: #64748b;
		--slate-600: #475569;
		--slate-700: #334155;
		--slate-800: #1e293b;
		--slate-900: #0f172a;
	}

	:global(*) {
		margin: 0;
		padding: 0;
		box-sizing: border-box;
	}

	:global(body) {
		font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
		background: var(--slate-100);
		color: var(--slate-800);
		font-size: 13px;
		line-height: 1.4;
		-webkit-print-color-adjust: exact;
		print-color-adjust: exact;
	}

	.page {
		max-width: 1280px;
		margin: 0 auto;
		padding: 8px 12px;
	}

	header {
		background: linear-gradient(135deg, var(--green-700), var(--green-800));
		color: white;
		padding: 12px 16px;
		border-radius: 8px;
		margin-bottom: 8px;
	}

	.header-inner {
		display: flex;
		align-items: center;
		gap: 12px;
	}

	.logo {
		font-size: 28px;
	}

	h1 {
		font-size: 18px;
		font-weight: 700;
		letter-spacing: -0.3px;
	}

	.subtitle {
		font-size: 11px;
		opacity: 0.85;
		margin-top: 1px;
	}

	.warnings {
		display: flex;
		flex-wrap: wrap;
		gap: 4px;
		margin-bottom: 6px;
	}

	.warning {
		padding: 4px 10px;
		border-radius: 6px;
		font-size: 12px;
		font-weight: 500;
		display: flex;
		align-items: center;
		gap: 4px;
	}

	.warning.error {
		background: #fef2f2;
		color: #991b1b;
		border: 1px solid #fecaca;
	}

	.warning.warn {
		background: #fffbeb;
		color: #92400e;
		border: 1px solid #fde68a;
	}

	.warning-icon {
		font-size: 14px;
	}

	.grid-main {
		display: grid;
		grid-template-columns: 300px 1fr;
		gap: 8px;
		align-items: start;
	}

	.card {
		background: white;
		border-radius: 8px;
		padding: 12px;
		border: 1px solid var(--slate-200);
	}

	.card h2 {
		font-size: 13px;
		font-weight: 600;
		color: var(--green-800);
		margin-bottom: 6px;
		padding-bottom: 4px;
		border-bottom: 2px solid var(--green-100);
	}

	.card h3 {
		font-size: 11px;
		font-weight: 600;
		color: var(--slate-500);
		text-transform: uppercase;
		letter-spacing: 0.5px;
		margin-top: 8px;
		margin-bottom: 4px;
	}

	.compact {
		padding: 10px 12px;
	}

	.inputs-card {
		position: sticky;
		top: 8px;
	}

	.input-group {
		display: flex;
		flex-direction: column;
		gap: 4px;
	}

	.input-group label {
		display: flex;
		flex-direction: column;
		gap: 1px;
	}

	.label-row {
		display: flex;
		justify-content: space-between;
		font-size: 11.5px;
		color: var(--slate-600);
	}

	.val {
		font-weight: 600;
		color: var(--green-700);
		font-variant-numeric: tabular-nums;
	}

	input[type="range"] {
		-webkit-appearance: none;
		appearance: none;
		width: 100%;
		height: 6px;
		border-radius: 3px;
		background: var(--slate-200);
		outline: none;
		cursor: pointer;
	}

	input[type="range"]::-webkit-slider-thumb {
		-webkit-appearance: none;
		appearance: none;
		width: 16px;
		height: 16px;
		border-radius: 50%;
		background: var(--green-600);
		border: 2px solid white;
		box-shadow: 0 1px 3px rgba(0,0,0,0.2);
		cursor: pointer;
	}

	input[type="range"]::-moz-range-thumb {
		width: 16px;
		height: 16px;
		border-radius: 50%;
		background: var(--green-600);
		border: 2px solid white;
		box-shadow: 0 1px 3px rgba(0,0,0,0.2);
		cursor: pointer;
	}

	input[type="range"]:hover::-webkit-slider-thumb {
		background: var(--green-500);
		transform: scale(1.1);
	}

	.results-col {
		display: flex;
		flex-direction: column;
		gap: 8px;
	}

	/* VERDICT */
	.verdict-card {
		text-align: center;
		padding: 14px;
	}

	.verdict-card.positive {
		background: linear-gradient(135deg, var(--green-50), var(--green-100));
		border-color: var(--green-300);
	}

	.verdict-card.negative {
		background: linear-gradient(135deg, #fef2f2, #fee2e2);
		border-color: #fecaca;
	}

	.verdict-header {
		display: flex;
		align-items: center;
		justify-content: center;
		gap: 10px;
	}

	.verdict-label {
		font-size: 14px;
		font-weight: 600;
		color: var(--green-800);
	}

	.verdict-card.negative .verdict-label {
		color: #991b1b;
	}

	.verdict-amount {
		font-size: 26px;
		font-weight: 700;
		color: var(--green-700);
		font-variant-numeric: tabular-nums;
	}

	.verdict-card.negative .verdict-amount {
		color: #dc2626;
	}

	.verdict-sub {
		font-size: 11px;
		color: var(--slate-500);
		margin-top: 2px;
	}

	.verdict-breakeven {
		font-size: 11px;
		margin-top: 4px;
		color: var(--slate-600);
		background: rgba(255,255,255,0.6);
		display: inline-block;
		padding: 2px 8px;
		border-radius: 4px;
	}

	/* QUOTITEIT */
	.row-between {
		display: flex;
		justify-content: space-between;
		align-items: center;
	}

	.metric-label {
		font-weight: 600;
		font-size: 13px;
	}

	.metric-value {
		font-size: 20px;
		font-weight: 700;
		font-variant-numeric: tabular-nums;
	}

	.bar-track {
		height: 10px;
		background: var(--slate-100);
		border-radius: 5px;
		margin: 4px 0;
		position: relative;
		overflow: visible;
	}

	.bar-fill {
		height: 100%;
		border-radius: 5px;
		transition: width 0.3s, background 0.3s;
	}

	.bar-markers {
		position: absolute;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
	}

	.bar-marker {
		position: absolute;
		top: -2px;
		width: 1px;
		height: 14px;
		background: var(--slate-400);
	}

	.bar-marker span {
		position: absolute;
		top: 16px;
		left: -10px;
		font-size: 9px;
		color: var(--slate-400);
	}

	.sub-metrics {
		font-size: 11px;
		color: var(--slate-500);
		margin-top: 8px;
	}

	/* MONTHLY COMPARE */
	.month-compare {
		display: grid;
		grid-template-columns: 1fr 1fr;
		gap: 10px;
	}

	.month-col {
		text-align: center;
	}

	.month-title {
		font-weight: 600;
		font-size: 12px;
		color: var(--slate-600);
		margin-bottom: 2px;
	}

	.month-amount {
		font-size: 22px;
		font-weight: 700;
		color: var(--green-700);
		font-variant-numeric: tabular-nums;
	}

	.month-col:last-child .month-amount {
		color: var(--slate-600);
	}

	.month-small {
		font-size: 11px;
		font-weight: 400;
		color: var(--slate-400);
	}

	.month-bar-stack {
		display: flex;
		height: 8px;
		border-radius: 4px;
		overflow: hidden;
		margin: 4px 0;
		gap: 1px;
	}

	.bar-seg {
		border-radius: 2px;
		min-width: 2px;
	}

	.month-details {
		text-align: left;
	}

	.detail-row {
		display: flex;
		justify-content: space-between;
		font-size: 11px;
		padding: 1px 0;
		color: var(--slate-600);
	}

	.detail-row.hra {
		color: var(--green-600);
		font-weight: 600;
	}

	/* WATERFALL */
	.waterfall {
		display: flex;
		flex-direction: column;
		gap: 3px;
	}

	.wf-row {
		display: grid;
		grid-template-columns: 180px 1fr;
		align-items: center;
		gap: 6px;
	}

	.wf-row.highlight {
		margin-top: 2px;
	}

	.wf-label {
		font-size: 11px;
		color: var(--slate-600);
		white-space: nowrap;
	}

	.wf-bar-wrap {
		height: 22px;
	}

	.wf-bar {
		height: 100%;
		border-radius: 4px;
		display: flex;
		align-items: center;
		padding: 0 6px;
		font-size: 11px;
		font-weight: 500;
		color: white;
		white-space: nowrap;
		min-width: fit-content;
	}

	.wf-bar.neg {
		background: var(--green-600);
	}

	.wf-bar.pos {
		background: var(--green-400);
	}

	.wf-bar.neutral {
		background: var(--slate-400);
	}

	.wf-bar.neutral-light {
		background: var(--slate-300);
	}

	.wf-divider {
		height: 1px;
		background: var(--slate-200);
		margin: 3px 0;
	}

	/* SCENARIOS */
	.scenario-chart {
		display: flex;
		flex-direction: column;
		gap: 3px;
	}

	.sc-row {
		display: grid;
		grid-template-columns: 40px 1fr 90px;
		align-items: center;
		gap: 4px;
	}

	.sc-label {
		font-size: 11px;
		font-weight: 600;
		color: var(--slate-600);
		text-align: right;
		font-variant-numeric: tabular-nums;
	}

	.sc-bars {
		display: flex;
		flex-direction: column;
		gap: 1px;
	}

	.sc-bar {
		height: 10px;
		border-radius: 3px;
		font-size: 9px;
		display: flex;
		align-items: center;
		padding: 0 4px;
		color: white;
		white-space: nowrap;
		min-width: fit-content;
	}

	.sc-bar.kopen {
		background: var(--green-500);
	}

	.sc-bar.kopen.profit {
		background: var(--green-400);
	}

	.sc-bar.huren {
		background: var(--slate-400);
	}

	.sc-verdict {
		font-size: 11px;
		font-weight: 700;
		text-align: right;
		font-variant-numeric: tabular-nums;
	}

	.sc-verdict.positive {
		color: var(--green-600);
	}

	.sc-verdict.negative {
		color: #dc2626;
	}

	.sc-legend {
		display: flex;
		gap: 12px;
		justify-content: center;
		margin-top: 4px;
		font-size: 10px;
		color: var(--slate-500);
	}

	.dot {
		display: inline-block;
		width: 8px;
		height: 8px;
		border-radius: 2px;
		margin-right: 3px;
		vertical-align: middle;
	}

	.dot.kopen {
		background: var(--green-500);
	}

	.dot.huren {
		background: var(--slate-400);
	}

	/* TABLE */
	table {
		width: 100%;
		border-collapse: collapse;
		font-size: 11px;
	}

	thead th {
		text-align: left;
		font-weight: 600;
		color: var(--slate-500);
		padding: 3px 4px;
		border-bottom: 2px solid var(--green-100);
		font-size: 10px;
		text-transform: uppercase;
		letter-spacing: 0.3px;
	}

	tbody td {
		padding: 3px 4px;
		border-bottom: 1px solid var(--slate-100);
		font-variant-numeric: tabular-nums;
	}

	.total-row td {
		border-top: 2px solid var(--green-200);
		font-weight: 700;
		color: var(--green-800);
	}

	.hra-cell {
		color: var(--green-600);
		font-weight: 600;
	}

	.table-note {
		font-size: 10px;
		color: var(--slate-400);
		margin-top: 4px;
		font-style: italic;
	}

	/* AFFORDABILITY */
	.afford-grid {
		display: grid;
		grid-template-columns: 1fr 1fr 1fr;
		gap: 8px;
	}

	.afford-item {
		text-align: center;
		padding: 6px;
		background: var(--slate-50);
		border-radius: 6px;
	}

	.afford-label {
		display: block;
		font-size: 10px;
		color: var(--slate-500);
		margin-bottom: 2px;
	}

	.afford-value {
		display: block;
		font-size: 18px;
		font-weight: 700;
		font-variant-numeric: tabular-nums;
		color: var(--green-700);
	}

	.afford-note {
		display: block;
		font-size: 9px;
		color: var(--slate-400);
	}

	/* FOOTER */
	footer {
		margin-top: 8px;
		padding: 6px 12px;
		text-align: center;
		font-size: 10px;
		color: var(--slate-400);
		border-top: 1px solid var(--slate-200);
	}

	/* PRINT */
	@media print {
		:global(body) {
			font-size: 10px;
			background: white;
		}

		.page {
			padding: 0;
			max-width: none;
		}

		.inputs-card {
			position: static;
		}

		.grid-main {
			grid-template-columns: 260px 1fr;
			gap: 6px;
		}

		.card {
			break-inside: avoid;
			border: 1px solid #ddd;
			padding: 8px;
		}

		header {
			padding: 8px 12px;
		}

		input[type="range"] {
			display: none;
		}

		.label-row {
			font-size: 10px;
		}

		.val {
			font-size: 10px;
		}
	}

	@media (max-width: 860px) {
		.grid-main {
			grid-template-columns: 1fr;
		}

		.inputs-card {
			position: static;
		}
	}
</style>
