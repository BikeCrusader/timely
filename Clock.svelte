<script>
  import { createEventDispatcher } from 'svelte';

  export let totalSeconds = 0;
  export let label = '';
  export let interactive = false;

  const dispatch = createEventDispatcher();

  const SIZE = 260;
  const CX = 130, CY = 130;
  const R = 108;

  // Okabe-Ito color-blind friendly palette
  const YEAR_COLOR   = '#E69F00'; // Orange
  const DAY_COLOR    = '#56B4E9'; // Sky Blue
  const HOUR_COLOR   = '#009E73'; // Bluish Green
  const MIN_COLOR    = '#0072B2'; // Blue
  const SEC_COLOR    = '#D55E00'; // Vermilion

  const SECS_PER_DAY  = 86400;
  // Use same 365-day year as App.svelte for consistency
  const SECS_PER_YEAR = 365 * SECS_PER_DAY;

  $: ts = totalSeconds ?? 0;

  // Decompose – handle negatives cleanly with modular arithmetic
  $: _sec  = ((ts % 60) + 60) % 60;
  $: _min  = ((Math.floor(ts / 60) % 60) + 60) % 60;
  $: _hour = ((Math.floor(ts / 3600) % 24) + 24) % 24;
  $: _day  = ((Math.floor(ts / SECS_PER_DAY) % 365) + 365) % 365;
  $: _year = Math.floor(ts / SECS_PER_YEAR);

  // Rotation angles in degrees (0° = 12 o'clock)
  $: secAngle  = (_sec / 60) * 360;
  $: minAngle  = ((_min + _sec / 60) / 60) * 360;
  $: hourAngle = ((_hour % 12 + _min / 60) / 12) * 360;
  $: dayAngle  = (_day / 365) * 360;
  $: yearAngle = (((_year % 10) + 10) % 10 / 10) * 360;

  // --- Static geometry (computed once) ---
  const ticks = Array.from({ length: 60 }, (_, i) => {
    const isMajor = i % 5 === 0;
    const outerR = R - 1;
    const innerR = isMajor ? R - 13 : R - 7;
    const rad = (i / 60) * 2 * Math.PI - Math.PI / 2;
    return {
      x1: CX + Math.cos(rad) * innerR,
      y1: CY + Math.sin(rad) * innerR,
      x2: CX + Math.cos(rad) * outerR,
      y2: CY + Math.sin(rad) * outerR,
      isMajor,
    };
  });

  const hourNumerals = [12,1,2,3,4,5,6,7,8,9,10,11].map((num, i) => {
    const rad = (i / 12) * 2 * Math.PI - Math.PI / 2;
    return { num, x: CX + Math.cos(rad) * (R - 26), y: CY + Math.sin(rad) * (R - 26) };
  });

  // --- Dragging via Pointer Events ---
  let svgEl;
  let draggingHand = null;

  function angleFromPointer(event) {
    const rect = svgEl.getBoundingClientRect();
    const scaleX = SIZE / rect.width;
    const scaleY = SIZE / rect.height;
    const dx = (event.clientX - rect.left) * scaleX - CX;
    const dy = (event.clientY - rect.top) * scaleY - CY;
    let a = Math.atan2(dy, dx) * 180 / Math.PI + 90;
    if (a < 0) a += 360;
    if (a >= 360) a -= 360;
    return a;
  }

  function startDrag(hand) {
    return (event) => {
      if (!interactive) return;
      draggingHand = hand;
      event.currentTarget.setPointerCapture(event.pointerId);
      event.preventDefault();
      event.stopPropagation();
    };
  }

  function handlePointerMove(event) {
    if (!draggingHand || !interactive) return;
    event.preventDefault();
    const angle = angleFromPointer(event);

    let h = Math.floor(_hour);
    let m = Math.floor(_min);
    let s = Math.round(_sec);

    if (draggingHand === 'hour') {
      const newHour12 = Math.round((angle / 360) * 12) % 12;
      const isPM = h >= 12;
      h = newHour12 + (isPM ? 12 : 0);
    } else if (draggingHand === 'minute') {
      m = Math.round((angle / 360) * 60) % 60;
    } else if (draggingHand === 'second') {
      s = Math.round((angle / 360) * 60) % 60;
    }

    dispatch('timechange',
      `${String(h).padStart(2,'0')}:${String(m).padStart(2,'0')}:${String(s).padStart(2,'0')}`
    );
  }

  function endDrag() {
    draggingHand = null;
  }
</script>

<div class="clock-wrapper">
  {#if label}
    <div class="clock-label">{label}</div>
  {/if}

  <svg
    bind:this={svgEl}
    viewBox="0 0 {SIZE} {SIZE}"
    class="clock-svg"
    role="img"
    aria-label="{label} clock"
    on:pointermove={handlePointerMove}
    on:pointerup={endDrag}
    on:pointercancel={endDrag}
  >
    <!-- Outer bezel -->
    <circle cx={CX} cy={CY} r={R + 6} fill="#f1f5f9" stroke="#cbd5e0" stroke-width="1.5"/>
    <!-- Face -->
    <circle cx={CX} cy={CY} r={R} fill="white"/>

    <!-- Tick marks -->
    {#each ticks as { x1, y1, x2, y2, isMajor }}
      <line {x1} {y1} {x2} {y2}
        stroke={isMajor ? '#64748b' : '#cbd5e0'}
        stroke-width={isMajor ? 2 : 1}
        stroke-linecap="round"
      />
    {/each}

    <!-- Hour numerals -->
    {#each hourNumerals as { num, x, y }}
      <text {x} {y} text-anchor="middle" dominant-baseline="central"
        font-size="11" fill="#475569" font-family="sans-serif" font-weight="600"
      >{num}</text>
    {/each}

    <!-- ── Year hand: very wide short fat stub ── -->
    <g transform="rotate({yearAngle}, {CX}, {CY})">
      <rect
        x={CX - 9} y={CY - R * 0.30}
        width="18" height={R * 0.35}
        rx="9"
        fill={YEAR_COLOR}
        opacity="0.9"
      />
    </g>

    <!-- ── Day hand: triangular arrow ── -->
    <g transform="rotate({dayAngle}, {CX}, {CY})">
      <polygon
        points="{CX},{CY - R * 0.50} {CX - 7},{CY + R * 0.10} {CX + 7},{CY + R * 0.10}"
        fill={DAY_COLOR}
        opacity="0.9"
      />
    </g>

    <!-- ── Hour hand: classic blunt rectangle (draggable) ── -->
    <g
      transform="rotate({hourAngle}, {CX}, {CY})"
      class={interactive ? 'hand-drag' : ''}
      on:pointerdown={startDrag('hour')}
      role={interactive ? 'button' : undefined}
      aria-label={interactive ? 'Hour hand – drag to set hour' : undefined}
    >
      <rect
        x={CX - 5} y={CY - R * 0.56}
        width="10" height={R * 0.68}
        rx="5"
        fill={HOUR_COLOR}
      />
    </g>

    <!-- ── Minute hand: longer, narrower rectangle (draggable) ── -->
    <g
      transform="rotate({minAngle}, {CX}, {CY})"
      class={interactive ? 'hand-drag' : ''}
      on:pointerdown={startDrag('minute')}
      role={interactive ? 'button' : undefined}
      aria-label={interactive ? 'Minute hand – drag to set minute' : undefined}
    >
      <rect
        x={CX - 3} y={CY - R * 0.80}
        width="6" height={R * 0.93}
        rx="3"
        fill={MIN_COLOR}
      />
    </g>

    <!-- ── Second hand: thin line + counterweight dot (draggable) ── -->
    <g
      transform="rotate({secAngle}, {CX}, {CY})"
      class={interactive ? 'hand-drag' : ''}
      on:pointerdown={startDrag('second')}
      role={interactive ? 'button' : undefined}
      aria-label={interactive ? 'Second hand – drag to set second' : undefined}
    >
      <line
        x1={CX} y1={CY + R * 0.26}
        x2={CX} y2={CY - R * 0.91}
        stroke={SEC_COLOR} stroke-width="1.5" stroke-linecap="round"
      />
      <circle cx={CX} cy={CY + R * 0.20} r="5" fill={SEC_COLOR}/>
    </g>

    <!-- Center cap -->
    <circle cx={CX} cy={CY} r="7" fill="#1e293b"/>
    <circle cx={CX} cy={CY} r="3.5" fill={SEC_COLOR}/>
  </svg>

  <!-- Legend -->
  <div class="clock-legend">
    <span class="leg" style="color:{YEAR_COLOR}">
      <svg width="12" height="12" viewBox="0 0 12 12" aria-hidden="true">
        <rect x="0" y="3" width="12" height="6" rx="3" fill={YEAR_COLOR}/>
      </svg>
      Year&nbsp;{_year}
    </span>
    <span class="leg" style="color:{DAY_COLOR}">
      <svg width="12" height="12" viewBox="0 0 12 12" aria-hidden="true">
        <polygon points="6,0 0,12 12,12" fill={DAY_COLOR}/>
      </svg>
      Day&nbsp;{_day}
    </span>
    <span class="leg" style="color:{HOUR_COLOR}">
      <svg width="12" height="12" viewBox="0 0 12 12" aria-hidden="true">
        <rect x="3" y="0" width="6" height="12" rx="3" fill={HOUR_COLOR}/>
      </svg>
      Hr&nbsp;{String(_hour).padStart(2,'0')}
    </span>
    <span class="leg" style="color:{MIN_COLOR}">
      <svg width="12" height="12" viewBox="0 0 12 12" aria-hidden="true">
        <rect x="4.5" y="0" width="3" height="12" rx="1.5" fill={MIN_COLOR}/>
      </svg>
      Min&nbsp;{String(Math.floor(_min)).padStart(2,'0')}
    </span>
    <span class="leg" style="color:{SEC_COLOR}">
      <svg width="12" height="12" viewBox="0 0 12 12" aria-hidden="true">
        <line x1="6" y1="0" x2="6" y2="9" stroke={SEC_COLOR} stroke-width="1.5" stroke-linecap="round"/>
        <circle cx="6" cy="11" r="2" fill={SEC_COLOR}/>
      </svg>
      Sec&nbsp;{String(Math.floor(_sec)).padStart(2,'0')}
    </span>
  </div>

  {#if interactive}
    <p class="drag-hint">Drag hour, minute, or second hands to set origin time</p>
  {/if}
</div>

<style>
  .clock-wrapper {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0.4rem;
  }

  .clock-label {
    font-size: 0.72rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.09em;
    color: #64748b;
  }

  .clock-svg {
    width: 100%;
    max-width: 220px;
    display: block;
    touch-action: none;
    user-select: none;
  }

  .hand-drag {
    cursor: grab;
  }
  .hand-drag:active {
    cursor: grabbing;
  }

  .clock-legend {
    display: flex;
    flex-wrap: wrap;
    gap: 0.25rem 0.55rem;
    justify-content: center;
    font-size: 0.68rem;
    font-weight: 700;
  }

  .leg {
    display: inline-flex;
    align-items: center;
    gap: 3px;
  }

  .drag-hint {
    font-size: 0.65rem;
    color: #94a3b8;
    font-style: italic;
    text-align: center;
    margin-top: 0.1rem;
  }
</style>
