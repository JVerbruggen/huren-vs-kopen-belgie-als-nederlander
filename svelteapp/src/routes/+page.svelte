<script lang="ts">
	// === TYPES ===
	type ExtraPost = { naam: string; bedrag: number; id: number };

	// === INPUTS (generic defaults, no personal data) ===
	let aankoopprijs = $state(250000);
	let eigenGeld = $state(25000);
	let hypotheekrente = $state(3.5);
	let looptijdJaren = $state(25);
	let verblijfsduurJaren = $state(3);
	let waardestijging = $state(3);
	let maandelijksehuur = $state(900);
	let gwePerMaand = $state(160);
	let huurServicekosten = $state(100);
	let huurIndexatie = $state(2.0);
	let brutoJaarinkomen = $state(55000);
	let nettoMaandinkomen = $state(3200);
	let vmeBijdrage = $state(150);
	let sbiBijdrage = $state(60);

	// Eenmalige kosten — editable with smart defaults
	let registratierechtenPct = $state(2);
	let notarisAankoop = $state(2500);
	let notarisKrediet = $state(2500);
	let bankDossier = $state(800);

	// Dynamic extra cost posts
	let nextId = $state(1);
	let extraGedeeldPosten = $state<ExtraPost[]>([]);
	let extraHuurPosten = $state<ExtraPost[]>([]);
	let extraKoopPosten = $state<ExtraPost[]>([]);

	const gedeeldTemplates = [
		{ naam: 'Internet + TV', bedrag: 55 },
		{ naam: 'Internet (alleen)', bedrag: 35 },
		{ naam: 'Afval (huisvuilzakken)', bedrag: 10 },
		{ naam: 'Mobiel abonnement', bedrag: 25 },
		{ naam: 'Aangepast', bedrag: 0 },
	];

	const huurTemplates = [
		{ naam: 'Garagebox', bedrag: 100 },
		{ naam: 'Berging', bedrag: 50 },
		{ naam: 'Parkeerplaats', bedrag: 75 },
		{ naam: 'Fietsenstalling', bedrag: 25 },
		{ naam: 'Aangepast', bedrag: 0 },
	];

	const koopTemplates = [
		{ naam: 'Garagebox (lening)', bedrag: 80 },
		{ naam: 'Onderhoudsreserve extra', bedrag: 50 },
		{ naam: 'Tuinonderhoud', bedrag: 40 },
		{ naam: 'Alarmsysteem', bedrag: 30 },
		{ naam: 'Aangepast', bedrag: 0 },
	];

	function addPost(list: ExtraPost[], template: { naam: string; bedrag: number }) {
		list.push({ naam: template.naam, bedrag: template.bedrag, id: nextId++ });
	}

	function removePost(list: ExtraPost[], id: number) {
		const idx = list.findIndex(p => p.id === id);
		if (idx >= 0) list.splice(idx, 1);
	}

	let showGedeeldMenu = $state(false);
	let showHuurMenu = $state(false);
	let showKoopMenu = $state(false);

	// === DERIVED CALCULATIONS ===
	let registratierechten = $derived(aankoopprijs * registratierechtenPct / 100);
	let bijkomendeKosten = $derived(registratierechten + notarisAankoop + notarisKrediet + bankDossier);
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

	// Eigenwoningforfait & HRA
	let eigenwoningforfait = $derived(aankoopprijs * 0.0035);
	let marginaalTarief = $derived(brutoJaarinkomen > 75518 ? 0.4950 : 0.3697);
	// HRA is capped at 36.97% even for higher incomes (2025+)
	let hraTarief = $derived(Math.min(marginaalTarief, 0.3697));

	function berekenRenteJaar(jaar: number): number {
		const r = maandRente;
		const n = aantalMaanden;
		let renteJaar = 0;
		for (let m = (jaar - 1) * 12; m < jaar * 12; m++) {
			const rs = hypotheekbedrag * (Math.pow(1 + r, n) - Math.pow(1 + r, m)) / (Math.pow(1 + r, n) - 1);
			renteJaar += rs * r;
		}
		return renteJaar;
	}

	let hraPerJaar = $derived(
		Array.from({ length: verblijfsduurJaren }, (_, i) => {
			const rente = berekenRenteJaar(i + 1);
			const nettoAftrek = rente - eigenwoningforfait;
			const teruggave = Math.max(0, nettoAftrek) * hraTarief;
			return { jaar: i + 1, rente, eigenwoningforfait, nettoAftrek, teruggave };
		})
	);

	let totaalHRA = $derived(hraPerJaar.reduce((s, j) => s + j.teruggave, 0));
	let maandelijkseHRA = $derived(totaalHRA / (verblijfsduurJaren * 12));

	// Restschuld
	let betaaldeMaanden = $derived(verblijfsduurJaren * 12);
	let restschuld = $derived(
		hypotheekbedrag * (Math.pow(1 + maandRente, aantalMaanden) - Math.pow(1 + maandRente, betaaldeMaanden)) /
		(Math.pow(1 + maandRente, aantalMaanden) - 1)
	);

	// Woningwaarde
	let woningwaardeNa = $derived(aankoopprijs * Math.pow(1 + waardestijging / 100, verblijfsduurJaren));
	let nettoVerkoopopbrengst = $derived(woningwaardeNa * 0.97);
	let nettoTerugontvangen = $derived(nettoVerkoopopbrengst - restschuld);

	// Koopkosten maand
	let brandverzekering = 25;
	let onroerendeVoorheffingMaand = $state(70);
	let onderhoudsreserve = $state(50);
	let extraKoopTotaal = $derived(extraKoopPosten.reduce((s, p) => s + p.bedrag, 0));

	let gedeeldKleuren = ['#0d9488', '#14b8a6', '#2dd4bf', '#5eead4', '#99f6e4'];

	let koopComponenten = $derived([
		{ naam: 'Hypotheek', bedrag: maandlastHypotheek, kleur: '#15803d' },
		{ naam: 'GWE', bedrag: gwePerMaand, kleur: '#22c55e' },
		{ naam: 'VME / syndic', bedrag: vmeBijdrage, kleur: '#4ade80' },
		{ naam: 'Schuldsaldoverzekering', bedrag: sbiBijdrage, kleur: '#86efac' },
		{ naam: 'Brandverzekering', bedrag: brandverzekering, kleur: '#a7f3d0' },
		{ naam: 'Onroerende voorheffing', bedrag: onroerendeVoorheffingMaand, kleur: '#6ee7b7' },
		{ naam: 'Onderhoudsreserve', bedrag: onderhoudsreserve, kleur: '#d1fae5' },
		...extraGedeeldPosten.map((p, i) => ({ naam: p.naam, bedrag: p.bedrag, kleur: gedeeldKleuren[i % gedeeldKleuren.length] })),
		...extraKoopPosten.map((p, i) => ({ naam: p.naam, bedrag: p.bedrag, kleur: ['#bbf7d0','#a3e8c0','#7dd3a8'][i % 3] })),
	]);

	let maandlastKopenBruto = $derived(koopComponenten.reduce((s, c) => s + c.bedrag, 0));
	let maandlastKopenNetto = $derived(maandlastKopenBruto - maandelijkseHRA);

	// Huurkosten maand (jaar 1)
	let extraHuurTotaal = $derived(extraHuurPosten.reduce((s, p) => s + p.bedrag, 0));

	let huurComponenten = $derived([
		{ naam: 'Kale huur', bedrag: maandelijksehuur, kleur: '#475569' },
		{ naam: 'GWE', bedrag: gwePerMaand, kleur: '#64748b' },
		{ naam: 'Servicekosten', bedrag: huurServicekosten, kleur: '#94a3b8' },
		...extraGedeeldPosten.map((p, i) => ({ naam: p.naam, bedrag: p.bedrag, kleur: gedeeldKleuren[i % gedeeldKleuren.length] })),
		...extraHuurPosten.map((p, i) => ({ naam: p.naam, bedrag: p.bedrag, kleur: ['#cbd5e1','#b0bec5','#9ea8b4'][i % 3] })),
	]);

	let maandlastHurenJaar1 = $derived(huurComponenten.reduce((s, c) => s + c.bedrag, 0));

	// Totale huurkosten met jaarlijkse indexatie
	let totaalHuurMetIndexatie = $derived((() => {
		let totaal = 0;
		for (let j = 0; j < verblijfsduurJaren; j++) {
			totaal += maandlastHurenJaar1 * Math.pow(1 + huurIndexatie / 100, j) * 12;
		}
		return totaal;
	})());

	// Gemiddelde maandlast huren over periode
	let gemiddeldeHuurMaand = $derived(totaalHuurMetIndexatie / (verblijfsduurJaren * 12));

	// Totale kosten
	let totaalKopen = $derived(maandlastKopenNetto * 12 * verblijfsduurJaren + bijkomendeKosten);
	let nettoKostenKopen = $derived(totaalKopen - nettoTerugontvangen);
	let opportuniteitskosten = $derived(eigenGeld * 0.04 * verblijfsduurJaren);
	let nettoKostenHuren = $derived(totaalHuurMetIndexatie + opportuniteitskosten);
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

	// Scenario's (only kopen varies, huren is fixed)
	let scenarios = $derived(
		[-3, -1, 0, 1, 3, 5, 7].map(pct => {
			const ww = aankoopprijs * Math.pow(1 + pct / 100, verblijfsduurJaren);
			const nvo = ww * 0.97;
			const nto = nvo - restschuld;
			const nkk = totaalKopen - nto;
			return { pct, nkk, voordeel: nettoKostenHuren - nkk };
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

	let quotiteitKleur = $derived(quotiteit <= 0.80 ? '#22c55e' : quotiteit <= 0.90 ? '#f59e0b' : '#ef4444');

	// Info tooltip state
	let activeTooltip = $state<string | null>(null);
	function toggleTip(id: string) {
		activeTooltip = activeTooltip === id ? null : id;
	}

	function fmt(n: number, decimals = 0): string {
		return n.toLocaleString('nl-NL', { minimumFractionDigits: decimals, maximumFractionDigits: decimals });
	}
	function eur(n: number): string {
		return '\u20AC' + fmt(n);
	}
</script>

<svelte:head>
	<title>Woning Kopen in België — Calculator</title>
	<link rel="preconnect" href="https://fonts.googleapis.com">
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous">
	<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
</svelte:head>

<!-- svelte-ignore a11y_click_events_have_key_events -->
<!-- svelte-ignore a11y_no_static_element_interactions -->
<div class="page" onclick={() => { showGedeeldMenu = false; showHuurMenu = false; showKoopMenu = false; activeTooltip = null; }}>
	<header>
		<div class="header-inner">
			<div class="logo">🏠</div>
			<div>
				<h1>Woning Kopen in België</h1>
				<p class="subtitle">Financiële calculator voor Nederlanders — kopen vs. huren in Vlaanderen</p>
			</div>
		</div>
	</header>

	<!-- Warnings: fixed height container so sliders don't jump -->
	<div class="warnings-container">
		{#each warnings as w}
			<div class="warning {w.type}">
				<span class="warning-icon">{w.type === 'error' ? '⚠' : '⚡'}</span>
				{w.msg}
			</div>
		{/each}
	</div>

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
					<span class="label-row">
						<span>Verblijfsduur
							<button class="info-btn" onclick={(e) => { e.stopPropagation(); toggleTip('verblijf'); }}>i</button>
						</span>
						<span class="val">{verblijfsduurJaren} jaar</span>
					</span>
					{#if activeTooltip === 'verblijf'}
						<div class="tooltip">Minimaal 1 jaar domicilie vereist voor 2% registratierechten. Bij &lt;3 jaar verhoogd risico op naheffing + 20% boete.</div>
					{/if}
					<input type="range" min="1" max="10" step="1" bind:value={verblijfsduurJaren}>
				</label>
				<label>
					<span class="label-row"><span>Verwachte waardestijging</span><span class="val">{fmt(waardestijging, 1)}%/jr</span></span>
					<input type="range" min="-5" max="10" step="0.5" bind:value={waardestijging}>
				</label>
			</div>

			<h3>Gedeelde kosten</h3>
			<div class="input-group">
				<label>
					<span class="label-row">
						<span>GWE (gas/water/elektra)
							<button class="info-btn" onclick={(e) => { e.stopPropagation(); toggleTip('gwe'); }}>i</button>
						</span>
						<span class="val">{eur(gwePerMaand)}</span>
					</span>
					{#if activeTooltip === 'gwe'}
						<div class="tooltip">
							Gas, water en elektriciteit. Richtbedragen per maand (1 persoon, Vlaanderen 2025):<br>
							<strong>Met gas:</strong> €130–200 (verwarming + koken op gas)<br>
							<strong>Zonder gas:</strong> €100–160 (warmtepomp of elektrisch)<br>
							<strong>Per extra persoon:</strong> +€30–50/mnd<br>
							Sterk afhankelijk van EPC-label en isolatie.
						</div>
					{/if}
					<input type="range" min="80" max="350" step="5" bind:value={gwePerMaand}>
				</label>
				{#each extraGedeeldPosten as post (post.id)}
					<div class="extra-post">
						<input type="text" class="extra-name" bind:value={post.naam} placeholder="Naam">
						<div class="extra-slider-row">
							<input type="range" min="0" max="500" step="5" bind:value={post.bedrag}>
							<span class="val extra-val">{eur(post.bedrag)}</span>
							<button class="remove-btn" onclick={() => removePost(extraGedeeldPosten, post.id)}>×</button>
						</div>
					</div>
				{/each}
				<div class="add-post-wrap">
					<button class="add-btn" onclick={(e) => { e.stopPropagation(); showGedeeldMenu = !showGedeeldMenu; showHuurMenu = false; showKoopMenu = false; }}>+ Extra gedeelde post</button>
					{#if showGedeeldMenu}
						<div class="template-menu">
							{#each gedeeldTemplates as t}
								<button class="template-item" onclick={() => { addPost(extraGedeeldPosten, t); showGedeeldMenu = false; }}>{t.naam}{t.bedrag ? ` (€${t.bedrag})` : ''}</button>
							{/each}
						</div>
					{/if}
				</div>
			</div>

			<h3>Huurkosten (maandelijks)</h3>
			<div class="input-group">
				<label>
					<span class="label-row"><span>Kale huur</span><span class="val">{eur(maandelijksehuur)}</span></span>
					<input type="range" min="500" max="2500" step="25" bind:value={maandelijksehuur}>
				</label>
				<label>
					<span class="label-row"><span>Servicekosten</span><span class="val">{eur(huurServicekosten)}</span></span>
					<input type="range" min="0" max="300" step="5" bind:value={huurServicekosten}>
				</label>
				<label>
					<span class="label-row">
						<span>Jaarlijkse huurindexatie
							<button class="info-btn" onclick={(e) => { e.stopPropagation(); toggleTip('huuridx'); }}>i</button>
						</span>
						<span class="val">{fmt(huurIndexatie, 1)}%</span>
					</span>
					{#if activeTooltip === 'huuridx'}
						<div class="tooltip">Huur wordt jaarlijks geïndexeerd op basis van de gezondheidsindex. Historisch gemiddeld ~2% in België.</div>
					{/if}
					<input type="range" min="0" max="5" step="0.1" bind:value={huurIndexatie}>
				</label>
				{#each extraHuurPosten as post (post.id)}
					<div class="extra-post">
						<input type="text" class="extra-name" bind:value={post.naam} placeholder="Naam">
						<div class="extra-slider-row">
							<input type="range" min="0" max="500" step="5" bind:value={post.bedrag}>
							<span class="val extra-val">{eur(post.bedrag)}</span>
							<button class="remove-btn" onclick={() => removePost(extraHuurPosten, post.id)}>×</button>
						</div>
					</div>
				{/each}
				<div class="add-post-wrap">
					<button class="add-btn" onclick={(e) => { e.stopPropagation(); showHuurMenu = !showHuurMenu; showKoopMenu = false; }}>+ Extra huurpost</button>
					{#if showHuurMenu}
						<div class="template-menu">
							{#each huurTemplates as t}
								<button class="template-item" onclick={() => { addPost(extraHuurPosten, t); showHuurMenu = false; }}>{t.naam}{t.bedrag ? ` (€${t.bedrag})` : ''}</button>
							{/each}
						</div>
					{/if}
				</div>
			</div>

			<h3>Koopkosten (maandelijks)</h3>
			<div class="input-group">
				<label>
					<span class="label-row">
						<span>VME-bijdrage
							<button class="info-btn" onclick={(e) => { e.stopPropagation(); toggleTip('vme'); }}>i</button>
						</span>
						<span class="val">{eur(vmeBijdrage)}</span>
					</span>
					{#if activeTooltip === 'vme'}
						<div class="tooltip">Verplicht bij appartementen. Dekt gemeenschappelijk onderhoud, verzekering gebouw, reservefonds. Vraag exact bedrag op vóór bod.</div>
					{/if}
					<input type="range" min="0" max="400" step="10" bind:value={vmeBijdrage}>
				</label>
				<label>
					<span class="label-row">
						<span>Schuldsaldoverzekering
							<button class="info-btn" onclick={(e) => { e.stopPropagation(); toggleTip('sbi'); }}>i</button>
						</span>
						<span class="val">{eur(sbiBijdrage)}</span>
					</span>
					{#if activeTooltip === 'sbi'}
						<div class="tooltip">Verplicht bij Belgische hypotheek. Aflopende overlijdensverzekering. Vergelijk min. 3 offertes; niet aftrekbaar in NL.</div>
					{/if}
					<input type="range" min="20" max="150" step="5" bind:value={sbiBijdrage}>
				</label>
				<label>
					<span class="label-row">
						<span>Onroerende voorheffing
							<button class="info-btn" onclick={(e) => { e.stopPropagation(); toggleTip('ovh'); }}>i</button>
						</span>
						<span class="val">{eur(onroerendeVoorheffingMaand)}</span>
					</span>
					{#if activeTooltip === 'ovh'}
						<div class="tooltip">Jaarlijkse belasting op onroerend goed, geïnd door Vlabel. Gebaseerd op geïndexeerd kadastraal inkomen × regionaal tarief × gemeentelijk opcentiemen. Gemeentebelasting zit hier inbegrepen. Typisch €800–1.200/jaar voor een appartement van €250k.</div>
					{/if}
					<input type="range" min="30" max="200" step="5" bind:value={onroerendeVoorheffingMaand}>
				</label>
				<label>
					<span class="label-row">
						<span>Onderhoudsreserve
							<button class="info-btn" onclick={(e) => { e.stopPropagation(); toggleTip('onderhoud'); }}>i</button>
						</span>
						<span class="val">{eur(onderhoudsreserve)}</span>
					</span>
					{#if activeTooltip === 'onderhoud'}
						<div class="tooltip">Privé-onderhoud van je wooneenheid (exclusief VME-gemeenschappelijk). Denk aan: schilderwerk, keukenapparatuur, sanitair, cv-ketel onderhoud. Vuistregel: 1% van woningwaarde per jaar ÷ 12. Bij een woning van €250k ≈ €200/mnd reservering, maar eerste jaren vaak lager.</div>
					{/if}
					<input type="range" min="0" max="200" step="5" bind:value={onderhoudsreserve}>
				</label>
				{#each extraKoopPosten as post (post.id)}
					<div class="extra-post">
						<input type="text" class="extra-name" bind:value={post.naam} placeholder="Naam">
						<div class="extra-slider-row">
							<input type="range" min="0" max="500" step="5" bind:value={post.bedrag}>
							<span class="val extra-val">{eur(post.bedrag)}</span>
							<button class="remove-btn" onclick={() => removePost(extraKoopPosten, post.id)}>×</button>
						</div>
					</div>
				{/each}
				<div class="add-post-wrap">
					<button class="add-btn" onclick={(e) => { e.stopPropagation(); showKoopMenu = !showKoopMenu; showHuurMenu = false; }}>+ Extra kooppost</button>
					{#if showKoopMenu}
						<div class="template-menu">
							{#each koopTemplates as t}
								<button class="template-item" onclick={() => { addPost(extraKoopPosten, t); showKoopMenu = false; }}>{t.naam}{t.bedrag ? ` (€${t.bedrag})` : ''}</button>
							{/each}
						</div>
					{/if}
				</div>
			</div>

			<h3>Inkomen</h3>
			<div class="input-group">
				<label>
					<span class="label-row">
						<span>Bruto jaarinkomen
							<button class="info-btn" onclick={(e) => { e.stopPropagation(); toggleTip('bruto'); }}>i</button>
						</span>
						<span class="val">{eur(brutoJaarinkomen)}</span>
					</span>
					{#if activeTooltip === 'bruto'}
						<div class="tooltip">Bepaalt je marginaal belastingtarief voor de hypotheekrenteaftrek. Boven €75.518: 49,50% (maar HRA geplafonneerd op 36,97%).</div>
					{/if}
					<input type="range" min="20000" max="150000" step="1000" bind:value={brutoJaarinkomen}>
				</label>
				<label>
					<span class="label-row"><span>Netto maandinkomen</span><span class="val">{eur(nettoMaandinkomen)}</span></span>
					<input type="range" min="1500" max="10000" step="50" bind:value={nettoMaandinkomen}>
				</label>
			</div>
		</section>

		<!-- RIGHT: RESULTS -->
		<div class="results-col">

			<!-- BETAALBAARHEID (top) -->
			<section class="card compact">
				<h2>
					Betaalbaarheid
					<button class="info-btn" onclick={(e) => { e.stopPropagation(); toggleTip('betaalbaar'); }}>i</button>
				</h2>
				{#if activeTooltip === 'betaalbaar'}
					<div class="tooltip">Belgische banken hanteren doorgaans max 1/3 van netto inkomen als hypotheeklast. Resterend inkomen moet leefkosten dekken.</div>
				{/if}
				<div class="afford-grid">
					<div class="afford-item">
						<span class="afford-label">Hypotheek % netto</span>
						<span class="afford-value" style="color: {maandlastHypotheek / nettoMaandinkomen > 0.35 ? '#ef4444' : maandlastHypotheek / nettoMaandinkomen > 0.33 ? '#f59e0b' : '#22c55e'}">{fmt(maandlastHypotheek / nettoMaandinkomen * 100, 1)}%</span>
						<span class="afford-note">Norm: max 33%</span>
					</div>
					<div class="afford-item">
						<span class="afford-label">Vrij na kopen</span>
						<span class="afford-value" style="color: {nettoMaandinkomen - maandlastKopenNetto < 500 ? '#ef4444' : '#22c55e'}">{eur(nettoMaandinkomen - maandlastKopenNetto)}</span>
						<span class="afford-note">per maand</span>
					</div>
					<div class="afford-item">
						<span class="afford-label">Vrij na huren</span>
						<span class="afford-value">{eur(nettoMaandinkomen - maandlastHurenJaar1)}</span>
						<span class="afford-note">per maand</span>
					</div>
				</div>
			</section>

			<!-- VERDICT -->
			<section class="card verdict-card" class:positive={voordeelKopen > 0} class:negative={voordeelKopen <= 0}>
				<div class="verdict-header">
					<span class="verdict-label">{voordeelKopen > 0 ? 'Kopen is voordeliger' : 'Huren is voordeliger'}</span>
					<span class="verdict-amount">{eur(Math.abs(voordeelKopen))}</span>
				</div>
				<p class="verdict-sub">netto over {verblijfsduurJaren} jaar &bull; waardestijging {fmt(waardestijging, 1)}%/jr &bull; huurindexatie {fmt(huurIndexatie, 1)}%/jr</p>
				<div class="verdict-breakeven">
					Breakeven waardestijging: <strong>{fmt(breakeven, 1)}%/jaar</strong>
					<button class="info-btn" onclick={(e) => { e.stopPropagation(); toggleTip('breakeven'); }}>i</button>
				</div>
				{#if activeTooltip === 'breakeven'}
					<div class="tooltip">De minimale jaarlijkse waardestijging waarbij kopen goedkoper is dan huren. Historisch gemiddelde Vlaanderen: +3,2%/jaar.</div>
				{/if}
			</section>

			<!-- QUOTITEIT -->
			<section class="card compact">
				<div class="row-between">
					<span class="metric-label">
						Quotiteit
						<button class="info-btn" onclick={(e) => { e.stopPropagation(); toggleTip('quotiteit'); }}>i</button>
					</span>
					<span class="metric-value" style="color: {quotiteitKleur}">{fmt(quotiteit * 100, 1)}%</span>
				</div>
				{#if activeTooltip === 'quotiteit'}
					<div class="tooltip">≤ 80% premium tarief • 80-90% standaard • 90-100% renteopslag mogelijk • &gt;100% niet mogelijk voor bestaande woningen.</div>
				{/if}
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
						<div class="month-amount kopen">{eur(maandlastKopenNetto)}<span class="month-small">/mnd</span></div>
						<div class="month-bar-stack">
							{#each koopComponenten as c}
								<div class="bar-seg" style="flex: {c.bedrag}; background: {c.kleur}" title="{c.naam} {eur(c.bedrag)}"></div>
							{/each}
						</div>
						<div class="month-details">
							{#each koopComponenten as c}
								<div class="detail-row">
									<span><span class="color-dot" style="background:{c.kleur}"></span>{c.naam}</span>
									<span>{eur(c.bedrag)}</span>
								</div>
							{/each}
							<div class="detail-row hra">
								<span><span class="color-dot" style="background:#059669"></span>HRA teruggave</span>
								<span>-{eur(maandelijkseHRA)}</span>
							</div>
							<div class="detail-row total">
								<span>Netto</span>
								<span>{eur(maandlastKopenNetto)}</span>
							</div>
						</div>
					</div>
					<div class="month-col">
						<div class="month-title">Huren <span class="month-yr1">(jaar 1)</span></div>
						<div class="month-amount huren">{eur(maandlastHurenJaar1)}<span class="month-small">/mnd</span></div>
						<div class="month-bar-stack">
							{#each huurComponenten as c}
								<div class="bar-seg" style="flex: {c.bedrag}; background: {c.kleur}" title="{c.naam} {eur(c.bedrag)}"></div>
							{/each}
						</div>
						<div class="month-details">
							{#each huurComponenten as c}
								<div class="detail-row">
									<span><span class="color-dot" style="background:{c.kleur}"></span>{c.naam}</span>
									<span>{eur(c.bedrag)}</span>
								</div>
							{/each}
							<div class="detail-row total">
								<span>Totaal</span>
								<span>{eur(maandlastHurenJaar1)}</span>
							</div>
							{#if huurIndexatie > 0}
								<div class="detail-row indexed">
									<span>Jaar {verblijfsduurJaren} (geïndexeerd)</span>
									<span>{eur(maandlastHurenJaar1 * Math.pow(1 + huurIndexatie / 100, verblijfsduurJaren - 1))}</span>
								</div>
							{/if}
						</div>
					</div>
				</div>
			</section>

			<!-- NETTO KOSTENVERGELIJKING (as summary table, not bars) -->
			<section class="card compact">
				<h2>
					Netto kostenvergelijking over {verblijfsduurJaren} jaar
					<button class="info-btn" onclick={(e) => { e.stopPropagation(); toggleTip('nettovgl'); }}>i</button>
				</h2>
				{#if activeTooltip === 'nettovgl'}
					<div class="tooltip">Bij kopen bouw je vermogen op via aflossing en waardestijging. De netto kosten = wat je kwijt bent minus wat je terugkrijgt bij verkoop.</div>
				{/if}
				<div class="cost-summary">
					<div class="cost-block kopen-block">
						<div class="cost-block-title">Kopen</div>
						<div class="cost-line"><span>Maandlasten ({verblijfsduurJaren} jr)</span><span class="cost-neg">{eur(maandlastKopenNetto * 12 * verblijfsduurJaren)}</span></div>
						<div class="cost-line"><span>Eenmalige kosten</span><span class="cost-neg">{eur(bijkomendeKosten)}</span></div>
						<div class="cost-line"><span>Totaal uitgegeven</span><span class="cost-neg">{eur(totaalKopen)}</span></div>
						<div class="cost-line pos"><span>Terugontvangen bij verkoop</span><span class="cost-pos">+{eur(nettoTerugontvangen)}</span></div>
						<div class="cost-line result">
							<span>Netto kosten kopen</span>
							<span class="cost-result">{eur(nettoKostenKopen)}</span>
						</div>
					</div>
					<div class="cost-block huren-block">
						<div class="cost-block-title">Huren</div>
						<div class="cost-line"><span>Huurlasten ({verblijfsduurJaren} jr, geïndexeerd)</span><span class="cost-neg">{eur(totaalHuurMetIndexatie)}</span></div>
						<div class="cost-line"><span>Opportuniteitskosten spaargeld</span><span class="cost-neg">{eur(opportuniteitskosten)}</span></div>
						<div class="cost-line result">
							<span>Netto kosten huren</span>
							<span class="cost-result">{eur(nettoKostenHuren)}</span>
						</div>
					</div>
					<div class="cost-diff" class:positive={voordeelKopen > 0} class:negative={voordeelKopen <= 0}>
						Verschil: <strong>{voordeelKopen > 0 ? 'kopen' : 'huren'} {eur(Math.abs(voordeelKopen))} voordeliger</strong>
					</div>
				</div>
			</section>

			<!-- SCENARIO'S -->
			<section class="card compact">
				<h2>
					Scenario's waardestijging
					<button class="info-btn" onclick={(e) => { e.stopPropagation(); toggleTip('scenario'); }}>i</button>
				</h2>
				{#if activeTooltip === 'scenario'}
					<div class="tooltip">Toont netto kosten van kopen bij verschillende waardestijgingen. De huurkosten ({eur(nettoKostenHuren)}) zijn hierin constant. Groen = kopen voordeliger.</div>
				{/if}
				<div class="sc-ref">Netto kosten huren: {eur(nettoKostenHuren)}</div>
				<div class="scenario-chart">
					{#each scenarios as s}
						{@const maxBar = Math.max(...scenarios.map(x => Math.abs(x.nkk)))}
						<div class="sc-row">
							<span class="sc-label">{s.pct > 0 ? '+' : ''}{s.pct}%/jr</span>
							<div class="sc-bar-wrap">
								<div
									class="sc-bar"
									class:winner={s.voordeel > 0}
									class:loser={s.voordeel <= 0}
									style="width: {Math.max(Math.abs(s.nkk) / maxBar * 100, 8)}%"
								>
									{eur(s.nkk)}
								</div>
							</div>
							<span class="sc-verdict" class:positive={s.voordeel > 0} class:negative={s.voordeel <= 0}>
								{s.voordeel > 0 ? 'kopen +' : 'huren +'}{eur(Math.abs(s.voordeel))}
							</span>
						</div>
					{/each}
				</div>
			</section>

			<!-- HRA TABLE -->
			<section class="card compact">
				<h2>
					Hypotheekrenteaftrek (HRA)
					<button class="info-btn" onclick={(e) => { e.stopPropagation(); toggleTip('hra'); }}>i</button>
				</h2>
				{#if activeTooltip === 'hra'}
					<div class="tooltip">
						Als kwalificerend buitenlands belastingplichtige (90%+ inkomen in NL) heb je recht op HRA.<br>
						<strong>Betaalde rente:</strong> de rente die je aan de bank betaalt (daalt jaarlijks bij annuïtair).<br>
						<strong>Eigenwoningforfait:</strong> 0,35% van de woningwaarde — wordt van de aftrek afgehaald.<br>
						<strong>Netto aftrek:</strong> rente minus forfait — het bedrag waarover je belasting terugkrijgt.<br>
						<strong>Teruggave:</strong> netto aftrek × {fmt(hraTarief * 100, 2)}% (geplafonneerd tarief 2025+).
					</div>
				{/if}
				<table>
					<thead>
						<tr>
							<th>Jaar</th>
							<th>Betaalde rente</th>
							<th>Forfait (0,35%)</th>
							<th>Netto aftrek</th>
							<th>Teruggave</th>
						</tr>
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
				<p class="table-note">≈ {eur(maandelijkseHRA)}/maand via voorlopige teruggave (aanvragen via mijn.belastingdienst.nl)</p>
			</section>

			<!-- EENMALIGE KOSTEN (editable) -->
			<section class="card compact">
				<h2>
					Eenmalige aankoopkosten
					<button class="info-btn" onclick={(e) => { e.stopPropagation(); toggleTip('eenmalig'); }}>i</button>
				</h2>
				{#if activeTooltip === 'eenmalig'}
					<div class="tooltip">Bijkomende kosten zijn nooit meefinancierbaar — altijd uit eigen middelen vóór aktedag. Pas bedragen aan als je concrete offertes hebt.</div>
				{/if}
				<div class="editable-costs">
					<div class="ec-row">
						<span>Registratierechten ({registratierechtenPct}%)</span>
						<span class="ec-value">{eur(registratierechten)}</span>
					</div>
					<div class="ec-row">
						<span>Notaris aankoopakte</span>
						<input type="number" class="ec-input" bind:value={notarisAankoop} min="0" step="100">
					</div>
					<div class="ec-row">
						<span>Notaris kredietakte</span>
						<input type="number" class="ec-input" bind:value={notarisKrediet} min="0" step="100">
					</div>
					<div class="ec-row">
						<span>Bankdossier + schatting</span>
						<input type="number" class="ec-input" bind:value={bankDossier} min="0" step="50">
					</div>
					<div class="ec-row total-row-ec">
						<span><strong>Totaal bijkomend</strong></span>
						<span><strong>{eur(bijkomendeKosten)}</strong></span>
					</div>
				</div>
				<p class="table-note">Besparing t.o.v. 12% standaardtarief: {eur(aankoopprijs * 0.10)}</p>
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
	}

	:global(*) { margin: 0; padding: 0; box-sizing: border-box; }

	:global(body) {
		font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
		background: var(--slate-100);
		color: var(--slate-800);
		font-size: 13px;
		line-height: 1.4;
		-webkit-print-color-adjust: exact;
		print-color-adjust: exact;
	}

	.page { max-width: 1280px; margin: 0 auto; padding: 8px 12px; }

	header {
		background: linear-gradient(135deg, var(--green-700), var(--green-800));
		color: white;
		padding: 12px 16px;
		border-radius: 8px;
		margin-bottom: 4px;
	}

	.header-inner { display: flex; align-items: center; gap: 12px; }
	.logo { font-size: 28px; }
	h1 { font-size: 18px; font-weight: 700; letter-spacing: -0.3px; }
	.subtitle { font-size: 11px; opacity: 0.85; margin-top: 1px; }

	/* Warnings: always-present container with fixed min-height */
	.warnings-container {
		min-height: 28px;
		display: flex;
		flex-wrap: wrap;
		gap: 4px;
		align-items: center;
		padding: 2px 0;
	}

	.warning {
		padding: 3px 10px;
		border-radius: 6px;
		font-size: 11px;
		font-weight: 500;
		display: flex;
		align-items: center;
		gap: 4px;
	}

	.warning.error { background: #fef2f2; color: #991b1b; border: 1px solid #fecaca; }
	.warning.warn { background: #fffbeb; color: #92400e; border: 1px solid #fde68a; }
	.warning-icon { font-size: 13px; }

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
		display: flex;
		align-items: center;
		gap: 4px;
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

	.compact { padding: 10px 12px; }
	.inputs-card { position: sticky; top: 8px; max-height: calc(100vh - 16px); overflow-y: auto; }

	.input-group { display: flex; flex-direction: column; gap: 4px; }
	.input-group label { display: flex; flex-direction: column; gap: 1px; }

	.label-row {
		display: flex;
		justify-content: space-between;
		font-size: 11.5px;
		color: var(--slate-600);
		align-items: center;
	}

	.val {
		font-weight: 600;
		color: var(--green-700);
		font-variant-numeric: tabular-nums;
	}

	/* Info buttons */
	.info-btn {
		display: inline-flex;
		align-items: center;
		justify-content: center;
		width: 15px;
		height: 15px;
		border-radius: 50%;
		border: 1.5px solid var(--slate-300);
		background: var(--slate-50);
		color: var(--slate-400);
		font-size: 9px;
		font-weight: 700;
		font-style: italic;
		cursor: pointer;
		line-height: 1;
		flex-shrink: 0;
	}

	.info-btn:hover { border-color: var(--green-400); color: var(--green-600); background: var(--green-50); }

	.tooltip {
		background: var(--slate-800);
		color: white;
		font-size: 11px;
		padding: 6px 10px;
		border-radius: 6px;
		line-height: 1.5;
		margin: 2px 0 4px;
		position: relative;
		z-index: 10;
	}

	/* Slider */
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

	/* Extra posts */
	.extra-post {
		background: var(--slate-50);
		border: 1px solid var(--slate-200);
		border-radius: 6px;
		padding: 4px 6px;
	}

	.extra-name {
		border: none;
		background: transparent;
		font-size: 11px;
		color: var(--slate-700);
		font-family: inherit;
		width: 100%;
		outline: none;
		font-weight: 500;
	}

	.extra-slider-row {
		display: flex;
		align-items: center;
		gap: 6px;
	}

	.extra-slider-row input[type="range"] { flex: 1; }
	.extra-val { font-size: 11px; min-width: 45px; text-align: right; }

	.remove-btn {
		background: none;
		border: none;
		color: var(--slate-400);
		cursor: pointer;
		font-size: 16px;
		line-height: 1;
		padding: 0 2px;
	}

	.remove-btn:hover { color: #ef4444; }

	.add-post-wrap { position: relative; }

	.add-btn {
		background: none;
		border: 1px dashed var(--slate-300);
		border-radius: 6px;
		padding: 3px 8px;
		font-size: 11px;
		color: var(--slate-500);
		cursor: pointer;
		width: 100%;
		font-family: inherit;
	}

	.add-btn:hover { border-color: var(--green-400); color: var(--green-600); }

	.template-menu {
		position: absolute;
		top: 100%;
		left: 0;
		right: 0;
		background: white;
		border: 1px solid var(--slate-200);
		border-radius: 6px;
		box-shadow: 0 4px 12px rgba(0,0,0,0.1);
		z-index: 20;
		overflow: hidden;
	}

	.template-item {
		display: block;
		width: 100%;
		text-align: left;
		border: none;
		background: none;
		padding: 6px 10px;
		font-size: 11px;
		font-family: inherit;
		color: var(--slate-700);
		cursor: pointer;
	}

	.template-item:hover { background: var(--green-50); color: var(--green-800); }

	.results-col { display: flex; flex-direction: column; gap: 8px; }

	/* AFFORDABILITY */
	.afford-grid { display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 8px; }

	.afford-item {
		text-align: center;
		padding: 6px;
		background: var(--slate-50);
		border-radius: 6px;
	}

	.afford-label { display: block; font-size: 10px; color: var(--slate-500); margin-bottom: 2px; }

	.afford-value {
		display: block;
		font-size: 18px;
		font-weight: 700;
		font-variant-numeric: tabular-nums;
		color: var(--green-700);
	}

	.afford-note { display: block; font-size: 9px; color: var(--slate-400); }

	/* VERDICT */
	.verdict-card { text-align: center; padding: 14px; }
	.verdict-card.positive { background: linear-gradient(135deg, var(--green-50), var(--green-100)); border-color: var(--green-300); }
	.verdict-card.negative { background: linear-gradient(135deg, #fef2f2, #fee2e2); border-color: #fecaca; }
	.verdict-header { display: flex; align-items: center; justify-content: center; gap: 10px; }

	.verdict-label { font-size: 14px; font-weight: 600; color: var(--green-800); }
	.verdict-card.negative .verdict-label { color: #991b1b; }

	.verdict-amount { font-size: 26px; font-weight: 700; color: var(--green-700); font-variant-numeric: tabular-nums; }
	.verdict-card.negative .verdict-amount { color: #dc2626; }

	.verdict-sub { font-size: 11px; color: var(--slate-500); margin-top: 2px; }

	.verdict-breakeven {
		font-size: 11px;
		margin-top: 4px;
		color: var(--slate-600);
		background: rgba(255,255,255,0.6);
		display: inline-flex;
		align-items: center;
		gap: 4px;
		padding: 2px 8px;
		border-radius: 4px;
	}

	/* QUOTITEIT */
	.row-between { display: flex; justify-content: space-between; align-items: center; }
	.metric-label { font-weight: 600; font-size: 13px; display: flex; align-items: center; gap: 4px; }
	.metric-value { font-size: 20px; font-weight: 700; font-variant-numeric: tabular-nums; }

	.bar-track { height: 10px; background: var(--slate-100); border-radius: 5px; margin: 4px 0; position: relative; }
	.bar-fill { height: 100%; border-radius: 5px; transition: width 0.3s, background 0.3s; }
	.bar-markers { position: absolute; top: 0; left: 0; right: 0; bottom: 0; }

	.bar-marker { position: absolute; top: -2px; width: 1px; height: 14px; background: var(--slate-400); }
	.bar-marker span { position: absolute; top: 16px; left: -10px; font-size: 9px; color: var(--slate-400); }

	.sub-metrics { font-size: 11px; color: var(--slate-500); margin-top: 8px; }

	/* MONTHLY */
	.month-compare { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
	.month-col { text-align: center; }
	.month-title { font-weight: 600; font-size: 12px; color: var(--slate-600); margin-bottom: 2px; }
	.month-yr1 { font-weight: 400; font-size: 10px; color: var(--slate-400); }

	.month-amount { font-size: 22px; font-weight: 700; font-variant-numeric: tabular-nums; margin-bottom: 4px; }
	.month-amount.kopen { color: var(--green-700); }
	.month-amount.huren { color: var(--slate-600); }
	.month-small { font-size: 11px; font-weight: 400; color: var(--slate-400); }

	.month-bar-stack {
		display: flex;
		height: 8px;
		border-radius: 4px;
		overflow: hidden;
		margin: 4px 0;
		gap: 1px;
	}

	.bar-seg { border-radius: 2px; min-width: 2px; }

	.month-details { text-align: left; }

	.detail-row {
		display: flex;
		justify-content: space-between;
		font-size: 11px;
		padding: 1.5px 0;
		color: var(--slate-600);
		align-items: center;
	}

	.detail-row.hra { color: #059669; font-weight: 600; border-top: 1px dashed var(--slate-200); margin-top: 2px; padding-top: 3px; }
	.detail-row.total { font-weight: 700; color: var(--slate-800); border-top: 1px solid var(--slate-300); margin-top: 2px; padding-top: 3px; }
	.detail-row.indexed { color: var(--slate-400); font-style: italic; font-size: 10px; }

	.color-dot {
		display: inline-block;
		width: 8px;
		height: 8px;
		border-radius: 2px;
		margin-right: 5px;
		vertical-align: middle;
		flex-shrink: 0;
	}

	/* COST SUMMARY (replaces waterfall bars) */
	.cost-summary { display: flex; flex-direction: column; gap: 6px; }

	.cost-block {
		background: var(--slate-50);
		border-radius: 6px;
		padding: 8px 10px;
	}

	.kopen-block { border-left: 3px solid var(--green-600); }
	.huren-block { border-left: 3px solid var(--slate-400); }

	.cost-block-title { font-weight: 600; font-size: 12px; color: var(--slate-700); margin-bottom: 4px; }

	.cost-line {
		display: flex;
		justify-content: space-between;
		font-size: 11px;
		padding: 1.5px 0;
		color: var(--slate-600);
	}

	.cost-line.pos { color: var(--green-600); }
	.cost-neg { font-variant-numeric: tabular-nums; }
	.cost-pos { font-variant-numeric: tabular-nums; color: var(--green-600); font-weight: 500; }

	.cost-line.result {
		border-top: 1.5px solid var(--slate-300);
		margin-top: 3px;
		padding-top: 4px;
		font-weight: 700;
		color: var(--slate-800);
	}

	.cost-result { font-variant-numeric: tabular-nums; }

	.cost-diff {
		text-align: center;
		padding: 6px;
		border-radius: 6px;
		font-size: 12px;
	}

	.cost-diff.positive { background: var(--green-50); color: var(--green-800); border: 1px solid var(--green-200); }
	.cost-diff.negative { background: #fef2f2; color: #991b1b; border: 1px solid #fecaca; }

	/* SCENARIOS */
	.sc-ref { font-size: 10px; color: var(--slate-400); margin-bottom: 4px; font-style: italic; }

	.scenario-chart { display: flex; flex-direction: column; gap: 3px; }

	.sc-row {
		display: grid;
		grid-template-columns: 50px 1fr 130px;
		align-items: center;
		gap: 6px;
	}

	.sc-label { font-size: 11px; font-weight: 600; color: var(--slate-600); text-align: right; font-variant-numeric: tabular-nums; }

	.sc-bar-wrap { height: 18px; }

	.sc-bar {
		height: 100%;
		border-radius: 4px;
		font-size: 10px;
		display: flex;
		align-items: center;
		padding: 0 6px;
		color: white;
		white-space: nowrap;
		min-width: fit-content;
		font-variant-numeric: tabular-nums;
	}

	.sc-bar.winner { background: var(--green-500); }
	.sc-bar.loser { background: var(--slate-400); }

	.sc-verdict { font-size: 10px; font-weight: 600; font-variant-numeric: tabular-nums; }
	.sc-verdict.positive { color: var(--green-600); }
	.sc-verdict.negative { color: #dc2626; }

	/* TABLE */
	table { width: 100%; border-collapse: collapse; font-size: 11px; }

	thead th {
		text-align: left;
		font-weight: 600;
		color: var(--slate-500);
		padding: 3px 4px;
		border-bottom: 2px solid var(--green-100);
		font-size: 10px;
		letter-spacing: 0.3px;
	}

	tbody td {
		padding: 3px 4px;
		border-bottom: 1px solid var(--slate-100);
		font-variant-numeric: tabular-nums;
	}

	.total-row td { border-top: 2px solid var(--green-200); font-weight: 700; color: var(--green-800); }
	.hra-cell { color: var(--green-600); font-weight: 600; }
	.table-note { font-size: 10px; color: var(--slate-400); margin-top: 4px; font-style: italic; }

	/* EDITABLE COSTS */
	.editable-costs { display: flex; flex-direction: column; gap: 2px; }

	.ec-row {
		display: flex;
		align-items: center;
		justify-content: space-between;
		padding: 3px 0;
		font-size: 11px;
		color: var(--slate-600);
		gap: 8px;
	}

	.ec-value { font-variant-numeric: tabular-nums; font-weight: 500; }

	.ec-input {
		width: 80px;
		text-align: right;
		border: 1px solid var(--slate-200);
		border-radius: 4px;
		padding: 2px 6px;
		font-size: 11px;
		font-family: inherit;
		color: var(--slate-700);
		background: var(--slate-50);
	}

	.ec-input:focus { outline: none; border-color: var(--green-400); }


	.total-row-ec {
		border-top: 2px solid var(--green-200);
		margin-top: 2px;
		padding-top: 4px;
		font-weight: 700;
		color: var(--green-800);
	}

	footer {
		margin-top: 8px;
		padding: 6px 12px;
		text-align: center;
		font-size: 10px;
		color: var(--slate-400);
		border-top: 1px solid var(--slate-200);
	}

	@media print {
		:global(body) { font-size: 10px; background: white; }
		.page { padding: 0; max-width: none; }
		.inputs-card { position: static; max-height: none; overflow: visible; }
		.grid-main { grid-template-columns: 250px 1fr; gap: 6px; }
		.card { break-inside: avoid; border: 1px solid #ddd; padding: 8px; }
		header { padding: 8px 12px; }
		input[type="range"] { display: none; }
		.add-btn, .remove-btn, .info-btn { display: none; }
		.template-menu { display: none; }
		.tooltip { display: none; }
		.warnings-container { min-height: 0; }
	}

	@media (max-width: 860px) {
		.grid-main { grid-template-columns: 1fr; }
		.inputs-card { position: static; max-height: none; }
	}
</style>
