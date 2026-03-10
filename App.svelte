<script>
  import { onMount } from 'svelte';

  const UNITS = ['seconds', 'minutes', 'hours', 'days', 'years'];

  let originTime = '';

  onMount(() => {
    const now = new Date();
    const h = String(now.getHours()).padStart(2, '0');
    const m = String(now.getMinutes()).padStart(2, '0');
    const s = String(now.getSeconds()).padStart(2, '0');
    originTime = `${h}:${m}:${s}`;
  });

  let intervals = [];

  function addInterval() {
    intervals = [...intervals, { value: 1, unit: 'minutes' }];
  }

  function removeInterval(i) {
    intervals = intervals.filter((_, idx) => idx !== i);
  }

  function toSeconds(value, unit) {
    const v = parseFloat(value) || 0;
    switch (unit) {
      case 'seconds': return v;
      case 'minutes': return v * 60;
      case 'hours':   return v * 3600;
      case 'days':    return v * 86400;
      case 'years':   return v * 365 * 86400;
    }
    return 0;
  }

  $: originTotalSeconds = (() => {
    if (!originTime) return 0;
    const parts = originTime.split(':').map(Number);
    return (parts[0] || 0) * 3600 + (parts[1] || 0) * 60 + (parts[2] || 0);
  })();

  $: resultTotalSeconds = (() => {
    let t = originTotalSeconds;
    for (const iv of intervals) t += toSeconds(iv.value, iv.unit);
    return t;
  })();

  function formatTime(totalSec) {
    const SECS_PER_DAY = 86400;
    const SECS_PER_YEAR = 365 * SECS_PER_DAY;

    let remaining = totalSec;
    const years = Math.floor(remaining / SECS_PER_YEAR);
    remaining -= years * SECS_PER_YEAR;
    let days = Math.floor(remaining / SECS_PER_DAY);
    remaining -= days * SECS_PER_DAY;
    if (remaining < 0) { remaining += SECS_PER_DAY; days -= 1; }

    const h = Math.floor(remaining / 3600);
    const m = Math.floor((remaining % 3600) / 60);
    const s = Math.floor(remaining % 60);
    const timeStr = `${String(h).padStart(2,'0')}:${String(m).padStart(2,'0')}:${String(s).padStart(2,'0')}`;

    const parts = [];
    if (years !== 0) parts.push(`${years}yr`);
    if (days !== 0)  parts.push(`${days}d`);
    parts.push(timeStr);
    return parts.join(' ');
  }

  $: resultDisplay = formatTime(resultTotalSeconds);

  // Build timeline points: origin + one per interval (cumulative)
  $: timelinePoints = (() => {
    const pts = [{ label: 'Origin', seconds: originTotalSeconds }];
    let running = originTotalSeconds;
    intervals.forEach((iv, i) => {
      running += toSeconds(iv.value, iv.unit);
      pts.push({ label: `#${i + 1}`, seconds: running });
    });
    return pts;
  })();

  const GW = 500, GH = 100, PAD = 56;
  const CY = 52; // center y of the baseline

  $: gMin = Math.min(...timelinePoints.map(p => p.seconds));
  $: gMax = Math.max(...timelinePoints.map(p => p.seconds));
  $: gRange = gMax - gMin || 1;

  function ptX(sec) {
    return PAD + ((sec - gMin) / gRange) * (GW - PAD * 2);
  }
</script>

<main>
  <h1>
    <svg class="clock-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
      <circle cx="12" cy="12" r="10"/>
      <polyline points="12 6 12 12 16 14"/>
    </svg>
    Timely
  </h1>

  <section class="card">
    <label for="origin">Origin time</label>
    <input id="origin" type="time" step="1" bind:value={originTime} />
  </section>

  <section class="card intervals">
    <div class="intervals-header">
      <span>Intervals</span>
      <button class="add-btn" on:click={addInterval}>+ Add</button>
    </div>

    {#if intervals.length === 0}
      <p class="empty">No intervals — result equals origin time.</p>
    {/if}

    {#each intervals as interval, i}
      <div class="interval-row">
        <input
          class="interval-value"
          type="number"
          step="any"
          bind:value={interval.value}
          on:input={() => intervals = intervals}
        />
        <select bind:value={interval.unit} on:change={() => intervals = intervals}>
          {#each UNITS as unit}
            <option value={unit}>{unit}</option>
          {/each}
        </select>
        <button class="remove-btn" on:click={() => removeInterval(i)} aria-label="Remove">✕</button>
      </div>
    {/each}
  </section>

  {#if timelinePoints.length > 1}
  <section class="card graph-card">
    <div class="graph-header">Timeline</div>
    <svg viewBox="0 0 {GW} {GH}" class="graph-svg" role="img" aria-label="Timeline graph">
      <!-- baseline -->
      <line x1={PAD - 8} y1={CY} x2={GW - PAD + 8} y2={CY} stroke="#e2e8f0" stroke-width="2"/>

      <!-- colored segments between consecutive points -->
      {#each timelinePoints.slice(1) as pt, i}
        {@const prev = timelinePoints[i]}
        {@const x1 = ptX(prev.seconds)}
        {@const x2 = ptX(pt.seconds)}
        <line
          x1={x1} y1={CY}
          x2={x2} y2={CY}
          stroke={pt.seconds >= prev.seconds ? '#48bb78' : '#fc8181'}
          stroke-width="3"
          stroke-linecap="round"
        />
        <!-- arrowhead -->
        {#if Math.abs(x2 - x1) > 6}
          {@const dir = pt.seconds >= prev.seconds ? 1 : -1}
          <polygon
            points="{x2},{CY} {x2 - dir*8},{CY - 4} {x2 - dir*8},{CY + 4}"
            fill={pt.seconds >= prev.seconds ? '#48bb78' : '#fc8181'}
          />
        {/if}
      {/each}

      <!-- dots and labels -->
      {#each timelinePoints as pt, i}
        {@const x = ptX(pt.seconds)}
        {@const above = i % 2 === 0}
        <circle cx={x} cy={CY} r={i === 0 ? 7 : 5}
          fill={i === 0 ? '#4f46e5' : '#805ad5'}
          stroke="#fff" stroke-width="2"
        />
        <text
          x={x} y={above ? CY - 14 : CY + 24}
          text-anchor="middle"
          font-size="10"
          fill="#4a5568"
          font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif"
        >{formatTime(pt.seconds)}</text>
        <text
          x={x} y={above ? CY - 26 : CY + 36}
          text-anchor="middle"
          font-size="9"
          fill="#a0aec0"
          font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif"
        >{pt.label}</text>
      {/each}
    </svg>
  </section>
  {/if}

  <section class="card result">
    <span class="result-label">Result</span>
    <span class="result-value">{resultDisplay}</span>
  </section>
</main>

<style>
  :global(*, *::before, *::after) { box-sizing: border-box; margin: 0; padding: 0; }
  :global(body) {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
    background: #f0f4f8;
    min-height: 100vh;
    display: flex;
    justify-content: center;
    padding: 2rem 1rem;
  }

  main {
    width: 100%;
    max-width: 480px;
  }

  h1 {
    font-size: 2rem;
    font-weight: 700;
    color: #1a202c;
    margin-bottom: 1.5rem;
    letter-spacing: -0.5px;
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }

  .clock-icon {
    width: 1.75rem;
    height: 1.75rem;
    color: #4f46e5;
    flex-shrink: 0;
  }

  .card {
    background: #fff;
    border-radius: 12px;
    padding: 1.25rem 1.5rem;
    margin-bottom: 1rem;
    box-shadow: 0 1px 4px rgba(0,0,0,0.08);
  }

  label {
    display: block;
    font-size: 0.75rem;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.08em;
    color: #718096;
    margin-bottom: 0.5rem;
  }

  input[type="time"] {
    font-size: 1.5rem;
    font-weight: 600;
    color: #1a202c;
    border: none;
    outline: none;
    background: transparent;
    width: 100%;
  }

  .intervals-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 0.75rem;
    font-size: 0.75rem;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.08em;
    color: #718096;
  }

  .add-btn {
    background: #4f46e5;
    color: #fff;
    border: none;
    border-radius: 6px;
    padding: 0.3rem 0.75rem;
    font-size: 0.8rem;
    font-weight: 600;
    cursor: pointer;
    transition: background 0.15s;
  }
  .add-btn:hover { background: #4338ca; }

  .empty {
    color: #a0aec0;
    font-size: 0.875rem;
    font-style: italic;
  }

  .interval-row {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    margin-top: 0.5rem;
  }

  .interval-value {
    width: 90px;
    font-size: 1rem;
    font-weight: 500;
    color: #1a202c;
    border: 1.5px solid #e2e8f0;
    border-radius: 8px;
    padding: 0.4rem 0.6rem;
    outline: none;
    transition: border-color 0.15s;
  }
  .interval-value:focus { border-color: #4f46e5; }

  select {
    flex: 1;
    font-size: 1rem;
    color: #1a202c;
    border: 1.5px solid #e2e8f0;
    border-radius: 8px;
    padding: 0.4rem 0.6rem;
    outline: none;
    background: #fff;
    cursor: pointer;
    transition: border-color 0.15s;
  }
  select:focus { border-color: #4f46e5; }

  .remove-btn {
    background: none;
    border: none;
    color: #cbd5e0;
    font-size: 0.85rem;
    cursor: pointer;
    padding: 0.3rem;
    border-radius: 4px;
    transition: color 0.15s;
  }
  .remove-btn:hover { color: #e53e3e; }

  /* Graph */
  .graph-card {
    padding: 1.25rem 1rem 1rem;
  }

  .graph-header {
    font-size: 0.75rem;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.08em;
    color: #718096;
    margin-bottom: 0.5rem;
  }

  .graph-svg {
    width: 100%;
    display: block;
    overflow: visible;
  }

  .result {
    background: #4f46e5;
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
  }

  .result-label {
    font-size: 0.75rem;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.08em;
    color: rgba(255,255,255,0.7);
  }

  .result-value {
    font-size: 1.75rem;
    font-weight: 700;
    color: #fff;
    letter-spacing: -0.5px;
  }
</style>
