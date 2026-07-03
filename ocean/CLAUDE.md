# INFINITUM — Project Rules
# Read this entire file before writing any code.

## WHAT YOU ARE BUILDING
A single-file, infinitely looping cinematic scroll experience with TWO
independent video layers: three background scene videos (scroll-scrubbed)
and a separate fish layer (alpha-channel webm, always on top, positioned
independently via scroll — never scrubbed, always playing). No side nav,
no thumbnail rail — purely cinematic, centered text only.
No npm. No build tools. No Three.js. No React.
Served locally with: python3 -m http.server 8080

## HARD RULES — NEVER BREAK
1. ONE FILE ONLY — everything in index.html
2. NO THREE.JS, NO CANVAS, NO WEBGL
3. THREE background scroll tracks: #track-reef, #track-deep, #track-current
   — each scrubbed independently via getBoundingClientRect (see
   BACKGROUND SCRUB ENGINE below).
4. THE FISH LAYER IS SEPARATE AND NEVER SCRUBBED. It is a single
   <video class="fish-layer"> element, muted, loop, autoplay, playing
   fish-idle.webm continuously at natural playback speed — its VIDEO
   TIME is never touched by scroll. Only its CSS transform (position,
   scale, and a horizontal flip for direction) is driven by scroll
   progress, computed from a hand-authored array of waypoints:

   const fishPath = [
     { progress: 0.00, xPercent: 50, yPercent: 60, scale: 1.0, flip: false },
     { progress: 0.15, xPercent: 30, yPercent: 45, scale: 1.1, flip: true },
     { progress: 0.30, xPercent: 65, yPercent: 55, scale: 0.9, flip: false },
     { progress: 0.45, xPercent: 40, yPercent: 40, scale: 1.0, flip: true },
     { progress: 0.60, xPercent: 55, yPercent: 60, scale: 1.0, flip: false },
     { progress: 0.75, xPercent: 35, yPercent: 50, scale: 0.85, flip: true },
     { progress: 0.90, xPercent: 50, yPercent: 45, scale: 1.0, flip: false },
     { progress: 1.00, xPercent: 50, yPercent: 60, scale: 1.0, flip: false }
   ];

   On every scroll tick, compute overall page progress (total scrolled /
   total scrollable height across all three tracks combined), find the two
   surrounding waypoints, linearly interpolate xPercent/yPercent/scale
   between them, and apply as
   transform: translate(-50%,-50%) scaleX(flip?-1:1) scale(scale)
   with left/top set from the interpolated percentages.
5. AT THE MOMENT OF THE LOOP WRAP (see rule 10), briefly swap the fish
   layer's video src to fish-dart.webm for about 1 second (the fish darts
   forward as if diving into the loop), then swap back to fish-idle.webm
   on the new cycle. This is the one moment the fish's clip changes.
6. Always attach BOTH scroll listeners on every background track:
   window.addEventListener("scroll", requestTick, { passive: true })
   lenis.on("scroll", requestTick)
7. Never animate a track container itself — only animate children inside it
8. Never use GSAP pin:true AND CSS position:sticky on the same element
9. Never add will-change:transform or transform:translateZ(0) to any
   background <video> element — degrades quality on Retina displays.
   The fish-layer video MAY use will-change:transform since only its
   CSS transform animates, never its playback.
10. THE LOOP IS FORWARD-ONLY. When #track-current's scroll progress
    passes 0.995 while scrolling downward, instantly jump scrollTop back
    to 0 using lenis.scrollTo(0, { immediate: true }). Reset #track-reef's
    video currentTime to 0 in the same tick. Trigger the fish-dart swap
    from rule 5. Guard with a 600ms cooldown flag. Do NOT implement
    backward/upward looping.
11. LAZY LOAD every background video: preload="none" by default, load via
    IntersectionObserver when its track is within 1 viewport-height of
    view. Show that act's poster image until readyState >= 2. The fish
    layer's three clips (idle/turn/dart) load immediately on page load
    since they're tiny and always needed.
12. This is a standalone creative piece: NO case studies, NO client links,
    NO sidebar, NO thumbnail rail, NO functional navigation beyond a
    single "← shreya.design" corner link. Everything else is purely
    visual and centered text.
13. Only ONE piece of personal text appears — during the House beat within
    Coral Realm. The hero line (Surface) and closing line (Beyond) are
    atmospheric copy, not personal — see build prompt Steps 4-5.

## CDN IMPORTS — exact order, always
<link rel="stylesheet" href="https://unpkg.com/lenis@1.3.23/dist/lenis.css">
<script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.5/gsap.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.5/ScrollTrigger.min.js"></script>
<script src="https://unpkg.com/lenis@1.3.23/dist/lenis.min.js"></script>
<link href="https://fonts.googleapis.com/css2?family=Fraunces:ital,opsz,wght@0,9..144,300;0,9..144,400;0,9..144,500;1,9..144,400&family=Space+Grotesk:wght@400;500;600&display=swap" rel="stylesheet">

## LENIS SETUP
const lenis = new Lenis({ lerp: 0.075, smoothWheel: true, syncTouch: true });
lenis.on("scroll", ScrollTrigger.update);
gsap.ticker.add((time) => { lenis.raf(time * 1000); });
gsap.ticker.lagSmoothing(0);

## BACKGROUND SCRUB ENGINE
One reusable function, called three times (once per background track):

function bindScrubTrack(track, video, onProgress) {
  let ticking = false, duration = 0, initialized = false;
  const clamp = (v,lo,hi) => Math.min(Math.max(v,lo),hi);
  function update() {
    if (!initialized||!duration||!Number.isFinite(duration)) return;
    const total = track.offsetHeight - window.innerHeight;
    const rect = track.getBoundingClientRect();
    const passed = clamp(-rect.top, 0, total);
    const progress = total > 0 ? passed/total : 0;
    if (video.readyState >= 2)
      video.currentTime = clamp(progress*duration, 0, duration);
    onProgress(progress);
  }
  function requestTick() {
    if (ticking) return; ticking = true;
    requestAnimationFrame(() => { update(); ticking = false; });
  }
  function initScrub() {
    duration = video.duration;
    if (!duration||!Number.isFinite(duration)) return;
    video.pause(); video.currentTime = 0; initialized = true;
    window.addEventListener("scroll", requestTick, { passive:true });
    window.addEventListener("resize", requestTick);
    if (typeof lenis !== "undefined") lenis.on("scroll", requestTick);
    requestTick();
  }
  if (video.readyState >= 2) initScrub();
  else {
    video.addEventListener("loadedmetadata", initScrub, { once:true });
    video.addEventListener("loadeddata", initScrub, { once:true });
  }
  return { requestVideoLoad: () => video.load() };
}

## TEXT TIMING REFERENCE (internal only — no visible nav)

The stage table below is used only to time when centered text appears —
there is no clickable nav or thumbnail UI. Use combined scroll progress
across all three tracks (same calculation as the fish layer) to know when
each text moment should be visible:

const stages = [
  { name: "Surface",     track: "reef",    range: [0.00, 0.20] },
  { name: "Coral Realm", track: "reef",    range: [0.20, 1.00] },
  { name: "Blue Abyss",  track: "deep",    range: [0.00, 0.30] },
  { name: "Giants",      track: "deep",    range: [0.30, 0.65] },
  { name: "Ancient",     track: "deep",    range: [0.65, 1.00] },
  { name: "Twilight",    track: "current", range: [0.00, 0.30] },
  { name: "The Abyss",   track: "current", range: [0.30, 0.70] },
  { name: "Beyond",      track: "current", range: [0.70, 1.00] },
];

## COLORS
#04101A ink · #071C2C abyss (text card bg) · #EAF3F1 foam (all text)
rgba(234,243,241,0.45) muted foam · #E2784A coral (rare accent only)
#8FD3E8 glow (focus outlines only)

## FONTS
Fraunces — all display text (hero line, personal line, closing line, "INFINITUM" wordmark)
Space Grotesk — corner link only

## MOBILE — HARD RULES
14. Every <video> MUST have muted playsinline attributes, fish layer
    included.
15. object-fit: cover + object-position: center on background videos.
16. Fish layer sized in vw/vh-relative units so it scales sensibly on
    narrow viewports — verify it doesn't become tiny or oversized on a
    phone screen.
17. All centered text blocks must remain legible and comfortably sized
    on a narrow viewport — test the hero line, personal line, and closing
    line specifically on a phone width, since Fraunces at desktop sizing
    can wrap awkwardly on narrow screens if not scaled down.

## ALWAYS END THE SCRIPT WITH
ScrollTrigger.refresh();
