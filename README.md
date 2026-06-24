import { useState, useEffect, useRef, useCallback, useMemo } from "react";

// ═══════════════════════════════════════════════════════════════════════════════
// DESIGN TOKENS
// ═══════════════════════════════════════════════════════════════════════════════
const DARK = {
  bg0:     "#020408",
  bg1:     "#050d1a",
  bg2:     "#0a1628",
  bg3:     "#0f1f38",
  surface: "rgba(10,22,40,0.65)",
  border:  "rgba(193,127,62,0.18)",
  text:    "#f0f4ff",
  muted:   "rgba(240,244,255,0.45)",
  subtle:  "rgba(240,244,255,0.2)",
  gold:    "#c17f3e",
  violet:  "#7c5cbf",
  cyan:    "#4fc3f7",
  green:   "#43e89e",
  red:     "#f06a6a",
};
const LIGHT = {
  bg0:     "#f7f8fc",
  bg1:     "#eef0f8",
  bg2:     "#e4e7f4",
  bg3:     "#d8ddf0",
  surface: "rgba(255,255,255,0.75)",
  border:  "rgba(124,92,191,0.22)",
  text:    "#0a0e1a",
  muted:   "rgba(10,14,26,0.5)",
  subtle:  "rgba(10,14,26,0.2)",
  gold:    "#a0621e",
  violet:  "#5b3fa0",
  cyan:    "#0d7ab5",
  green:   "#1a9e65",
  red:     "#c0392b",
};

// ═══════════════════════════════════════════════════════════════════════════════
// UTILS
// ═══════════════════════════════════════════════════════════════════════════════
const clamp = (v, lo, hi) => Math.max(lo, Math.min(hi, v));
const rand  = (lo, hi)    => lo + Math.random() * (hi - lo);
const lerp  = (a, b, t)   => a + (b - a) * t;
const hex2rgb = hex => [
  parseInt(hex.slice(1,3),16),
  parseInt(hex.slice(3,5),16),
  parseInt(hex.slice(5,7),16),
];
const rgba = (hex, a) => { const [r,g,b] = hex2rgb(hex); return `rgba(${r},${g},${b},${a})`; };

function useIntersect(options = {}) {
  const ref = useRef(null);
  const [visible, setVisible] = useState(false);
  useEffect(() => {
    const el = ref.current;
    if (!el) return;
    const obs = new IntersectionObserver(([e]) => setVisible(e.isIntersecting), { threshold: 0.15, ...options });
    obs.observe(el);
    return () => obs.disconnect();
  }, []);
  return [ref, visible];
}

function useCountUp(target, visible, duration = 2000) {
  const [val, setVal] = useState(0);
  useEffect(() => {
    if (!visible) return;
    let start = null;
    const step = ts => {
      if (!start) start = ts;
      const p = Math.min((ts - start) / duration, 1);
      const ease = 1 - Math.pow(1 - p, 3);
      setVal(Math.round(ease * target));
      if (p < 1) requestAnimationFrame(step);
    };
    requestAnimationFrame(step);
  }, [visible, target, duration]);
  return val;
}

// ═══════════════════════════════════════════════════════════════════════════════
// ADVANCED 3D CANVAS — Neural Universe
// ═══════════════════════════════════════════════════════════════════════════════
function NeuralCanvas({ theme, mouseX, mouseY, scrollRatio }) {
  const canvasRef = useRef(null);
  const stRef     = useRef({ t: 0, particles: [], links: [], streams: [], sparks: [] });
  const rafRef    = useRef(null);

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;
    const ctx = canvas.getContext("2d");
    const resize = () => {
      canvas.width  = canvas.offsetWidth  * Math.min(devicePixelRatio, 2);
      canvas.height = canvas.offsetHeight * Math.min(devicePixelRatio, 2);
    };
    resize();
    const ro = new ResizeObserver(resize);
    ro.observe(canvas);

    const st = stRef.current;
    const N = Math.min(window.innerWidth < 768 ? 120 : 220, 220);

    // Spherical particle shell
    st.particles = Array.from({ length: N }, (_, i) => {
      const theta = rand(0, Math.PI * 2);
      const phi   = Math.acos(rand(-1, 1));
      const layer = Math.floor(rand(0, 3));
      const r     = [55, 100, 155][layer];
      return {
        ox: Math.sin(phi) * Math.cos(theta) * r,
        oy: Math.sin(phi) * Math.sin(theta) * r,
        oz: Math.cos(phi) * r,
        px: 0, py: 0,
        vx: rand(-0.08, 0.08), vy: rand(-0.08, 0.08), vz: rand(-0.08, 0.08),
        size:  rand(1.0, 3.2),
        speed: rand(0.2, 0.9),
        phase: rand(0, Math.PI * 2),
        layer,
        hue: ["gold","violet","cyan"][layer],
        pulse: rand(0, Math.PI * 2),
      };
    });

    // Vertical data streams
    st.streams = Array.from({ length: 40 }, () => ({
      x:     rand(-0.5, 0.5),
      y:     rand(-0.6, 0.2),
      len:   rand(0.05, 0.15),
      speed: rand(0.002, 0.006),
      alpha: rand(0.08, 0.35),
      width: rand(0.5, 1.5),
    }));

    // Floating sparks
    st.sparks = [];

    const COLORS = {
      gold:   "#c17f3e",
      violet: "#7c5cbf",
      cyan:   "#4fc3f7",
    };

    const draw = () => {
      const W = canvas.width, H = canvas.height;
      const cx = W / 2, cy = H / 2;
      st.t += 0.007;

      ctx.clearRect(0, 0, W, H);

      const mx = (mouseX - 0.5) * 70;
      const my = (mouseY - 0.5) * 50;
      const scroll3d = scrollRatio * 0.4;

      // ── data streams ──
      st.streams.forEach(s => {
        s.y += s.speed;
        if (s.y > 0.7) s.y = -0.7;
        const sx = cx + s.x * W;
        const sy = cy + s.y * H;
        const sl = s.len * H;
        const grad = ctx.createLinearGradient(sx, sy - sl, sx, sy);
        grad.addColorStop(0, `rgba(79,195,247,0)`);
        grad.addColorStop(0.6, `rgba(79,195,247,${s.alpha})`);
        grad.addColorStop(1, `rgba(79,195,247,0)`);
        ctx.beginPath();
        ctx.strokeStyle = grad;
        ctx.lineWidth = s.width;
        ctx.moveTo(sx, sy - sl);
        ctx.lineTo(sx, sy);
        ctx.stroke();
      });

      // ── project particles ──
      const fov = 380;
      const cosX = Math.cos(my * 0.013 + scroll3d);
      const sinX = Math.sin(my * 0.013 + scroll3d);
      const cosY = Math.cos(mx * 0.011);
      const sinY = Math.sin(mx * 0.011);

      const proj = st.particles.map(p => {
        const angle = st.t * p.speed + p.phase;
        const wobble = Math.sin(angle) * 6;
        let px = p.ox + wobble * Math.cos(p.phase);
        let py = p.oy + wobble * Math.sin(p.phase);
        let pz = p.oz + Math.cos(angle * 0.7) * 4;

        // Y-axis rotation (mouse X)
        const ry = px * cosY - pz * sinY;
        const rz0 = px * sinY + pz * cosY;
        // X-axis rotation (mouse Y + scroll)
        const ry2 = py * cosX - rz0 * sinX;
        const rz  = py * sinX + rz0 * cosX;

        const scale = fov / (fov + rz + 180);
        p.px = cx + ry * scale;
        p.py = cy + ry2 * scale;

        return { ...p, sx: p.px, sy: p.py, scale, depth: rz };
      }).sort((a, b) => a.depth - b.depth);

      // ── neural links ──
      for (let i = 0; i < proj.length; i++) {
        for (let j = i + 1; j < proj.length; j++) {
          if (proj[i].layer !== proj[j].layer && Math.random() > 0.003) continue;
          const dx = proj[i].sx - proj[j].sx;
          const dy = proj[i].sy - proj[j].sy;
          const d  = Math.sqrt(dx*dx + dy*dy);
          const maxD = proj[i].layer === proj[j].layer ? 75 : 50;
          if (d < maxD) {
            const alpha = (1 - d / maxD) * 0.28 * proj[i].scale;
            ctx.beginPath();
            ctx.strokeStyle = `rgba(124,92,191,${alpha})`;
            ctx.lineWidth = 0.6;
            ctx.moveTo(proj[i].sx, proj[i].sy);
            ctx.lineTo(proj[j].sx, proj[j].sy);
            ctx.stroke();
          }
        }
      }

      // ── draw particles ──
      proj.forEach(p => {
        const col = COLORS[p.hue];
        const [r, g, b] = hex2rgb(col);
        const alpha = clamp(p.scale * 1.6, 0, 1);
        const pulse = (Math.sin(st.t * 2 + p.pulse) + 1) * 0.5;
        const sz    = p.size * p.scale * (1 + pulse * 0.3);

        const grd = ctx.createRadialGradient(p.sx, p.sy, 0, p.sx, p.sy, sz * 3.5);
        grd.addColorStop(0,   `rgba(${r},${g},${b},${alpha})`);
        grd.addColorStop(0.4, `rgba(${r},${g},${b},${alpha * 0.5})`);
        grd.addColorStop(1,   `rgba(${r},${g},${b},0)`);
        ctx.beginPath();
        ctx.fillStyle = grd;
        ctx.arc(p.sx, p.sy, sz * 3.5, 0, Math.PI * 2);
        ctx.fill();
      });

      // ── central core glow ──
      const core = ctx.createRadialGradient(cx, cy, 0, cx, cy, 130 + Math.sin(st.t) * 15);
      core.addColorStop(0,   "rgba(193,127,62,0.15)");
      core.addColorStop(0.3, "rgba(124,92,191,0.08)");
      core.addColorStop(0.7, "rgba(79,195,247,0.04)");
      core.addColorStop(1,   "transparent");
      ctx.fillStyle = core;
      ctx.fillRect(0, 0, W, H);

      // ── rotating orbit rings ──
      const rings = [
        { r: 160, color: "rgba(193,127,62,0.12)", speed: 0.003,  dashes: [8, 16]  },
        { r: 220, color: "rgba(124,92,191,0.08)", speed: -0.002, dashes: [4, 24]  },
        { r: 290, color: "rgba(79,195,247,0.06)", speed: 0.0015, dashes: [12, 20] },
      ];
      rings.forEach(ring => {
        ctx.save();
        ctx.translate(cx, cy);
        ctx.rotate(st.t * ring.speed * 200);
        ctx.beginPath();
        ctx.strokeStyle = ring.color;
        ctx.lineWidth = 1;
        ctx.setLineDash(ring.dashes);
        ctx.arc(0, 0, ring.r, 0, Math.PI * 2);
        ctx.stroke();
        ctx.setLineDash([]);
        ctx.restore();
      });

      // ── scanline sweep ──
      const sweepY = ((st.t * 0.3) % 1) * H;
      const sweep = ctx.createLinearGradient(0, sweepY - 60, 0, sweepY + 60);
      sweep.addColorStop(0,   "transparent");
      sweep.addColorStop(0.5, "rgba(79,195,247,0.04)");
      sweep.addColorStop(1,   "transparent");
      ctx.fillStyle = sweep;
      ctx.fillRect(0, sweepY - 60, W, 120);

      rafRef.current = requestAnimationFrame(draw);
    };

    draw();
    return () => { cancelAnimationFrame(rafRef.current); ro.disconnect(); };
  // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  return (
    <canvas ref={canvasRef} style={{
      position: "absolute", inset: 0,
      width: "100%", height: "100%",
      pointerEvents: "none",
    }} />
  );
}

// ═══════════════════════════════════════════════════════════════════════════════
// HOLOGRAPHIC HUD ELEMENT
// ═══════════════════════════════════════════════════════════════════════════════
function HudCorner({ pos = "tl", T }) {
  const size = 20;
  const styles = {
    tl: { top: 0, left: 0, borderTop: `1px solid ${T.gold}`, borderLeft:  `1px solid ${T.gold}` },
    tr: { top: 0, right: 0,borderTop: `1px solid ${T.gold}`, borderRight: `1px solid ${T.gold}` },
    bl: { bottom:0,left: 0, borderBottom:`1px solid ${T.gold}`,borderLeft:`1px solid ${T.gold}` },
    br: { bottom:0,right:0, borderBottom:`1px solid ${T.gold}`,borderRight:`1px solid ${T.gold}`},
  };
  return (
    <div style={{
      position: "absolute", width: size, height: size,
      ...styles[pos],
    }} />
  );
}

function GlassCard({ children, style = {}, glow = false, hoverGlow = false, T, onClick }) {
  const [hov, setHov] = useState(false);
  const active = hov && hoverGlow;
  return (
    <div
      onClick={onClick}
      onMouseEnter={() => setHov(true)}
      onMouseLeave={() => setHov(false)}
      style={{
        position: "relative",
        background: T.surface,
        border: `1px solid ${active || glow ? rgba(T.gold, 0.45) : T.border}`,
        borderRadius: 16,
        backdropFilter: "blur(20px)",
        WebkitBackdropFilter: "blur(20px)",
        boxShadow: active || glow
          ? `0 0 40px ${rgba(T.gold, 0.15)}, 0 8px 32px rgba(0,0,0,0.3), inset 0 1px 0 rgba(255,255,255,0.07)`
          : `0 8px 32px rgba(0,0,0,0.2), inset 0 1px 0 rgba(255,255,255,0.04)`,
        transition: "all 0.35s cubic-bezier(0.22,1,0.36,1)",
        cursor: onClick ? "pointer" : "default",
        ...style,
      }}
    >
      <HudCorner pos="tl" T={T} />
      <HudCorner pos="tr" T={T} />
      <HudCorner pos="bl" T={T} />
      <HudCorner pos="br" T={T} />
      {children}
    </div>
  );
}

// ═══════════════════════════════════════════════════════════════════════════════
// REVEAL WRAPPER — intersection-triggered
// ═══════════════════════════════════════════════════════════════════════════════
function Reveal({ children, delay = 0, dir = "up", style = {} }) {
  const [ref, visible] = useIntersect();
  const offsets = { up: "40px", down: "-40px", left: "-40px", right: "40px" };
  const axis    = dir === "left" || dir === "right" ? "X" : "Y";
  return (
    <div
      ref={ref}
      style={{
        transform: visible ? "translate(0,0)" : `translate${axis}(${offsets[dir]})`,
        opacity:   visible ? 1 : 0,
        transition: `all 0.75s cubic-bezier(0.22,1,0.36,1) ${delay}ms`,
        ...style,
      }}
    >
      {children}
    </div>
  );
}

// ═══════════════════════════════════════════════════════════════════════════════
// SECTION LABEL (eyebrow)
// ═══════════════════════════════════════════════════════════════════════════════
function Eyebrow({ text, T }) {
  return (
    <div style={{
      display: "inline-flex", alignItems: "center", gap: 8,
      marginBottom: 20,
      padding: "6px 16px",
      background: rgba(T.gold, 0.1),
      border: `1px solid ${rgba(T.gold, 0.3)}`,
      borderRadius: 40,
      fontSize: 12,
      fontWeight: 600,
      color: T.gold,
      letterSpacing: "0.12em",
      textTransform: "uppercase",
    }}>
      <span style={{ width: 6, height: 6, borderRadius: "50%", background: T.gold, display: "inline-block", animation: "blink 1.4s ease-in-out infinite" }} />
      {text}
    </div>
  );
}

// ═══════════════════════════════════════════════════════════════════════════════
// NAV
// ═══════════════════════════════════════════════════════════════════════════════
const NAV_LINKS = [
  { label: "Capabilities", href: "#capabilities" },
  { label: "How It Works", href: "#how-it-works"  },
  { label: "Demos",        href: "#demos"          },
  { label: "Pricing",      href: "#pricing"        },
];

function Nav({ T, darkMode, setDarkMode, scrolled }) {
  const [menuOpen, setMenuOpen] = useState(false);
  const bgAlpha = scrolled ? 0.88 : 0;

  return (
    <>
      <nav
        role="navigation"
        aria-label="Primary navigation"
        style={{
          position: "fixed", top: 0, left: 0, right: 0, zIndex: 500,
          height: 64,
          display: "flex", alignItems: "center", justifyContent: "space-between",
          padding: "0 clamp(20px, 4vw, 56px)",
          background: scrolled ? (darkMode ? `rgba(5,13,26,${bgAlpha})` : `rgba(247,248,252,${bgAlpha})`) : "transparent",
          backdropFilter: scrolled ? "blur(24px)" : "none",
          WebkitBackdropFilter: scrolled ? "blur(24px)" : "none",
          borderBottom: scrolled ? `1px solid ${T.border}` : "none",
          transition: "all 0.4s ease",
        }}
      >
        {/* Logo */}
        <a href="#" aria-label="Claude AI home" style={{ display:"flex", alignItems:"center", gap:10, textDecoration:"none" }}>
          <div style={{
            width: 34, height: 34, borderRadius: 10,
            background: `linear-gradient(135deg, ${T.gold}, ${T.violet})`,
            display:"flex", alignItems:"center", justifyContent:"center",
            fontSize: 17, fontWeight: 800, color: "#fff",
            boxShadow: `0 0 20px ${rgba(T.gold, 0.4)}`,
          }}>C</div>
          <span style={{ color: T.text, fontWeight: 700, fontSize: 17, letterSpacing:"0.01em" }}>Claude</span>
          <span style={{
            fontSize: 10, fontWeight: 600, color: T.gold, letterSpacing:"0.1em",
            background: rgba(T.gold, 0.12), border: `1px solid ${rgba(T.gold,0.3)}`,
            borderRadius: 4, padding:"2px 6px", marginTop: 1,
          }}>AI</span>
        </a>

        {/* Desktop links */}
        <div style={{ display:"flex", gap:32, alignItems:"center" }}
             className="nav-desktop">
          {NAV_LINKS.map(l => (
            <a key={l.label} href={l.href} style={{
              color: T.muted, fontSize:14, textDecoration:"none",
              fontWeight: 500, letterSpacing:"0.03em",
              transition:"color 0.2s",
            }}
            onMouseEnter={e=>e.target.style.color=T.text}
            onMouseLeave={e=>e.target.style.color=T.muted}
            >{l.label}</a>
          ))}
        </div>

        {/* Right controls */}
        <div style={{ display:"flex", gap:12, alignItems:"center" }}>
          {/* Theme toggle */}
          <button
            aria-label={darkMode ? "Switch to light mode" : "Switch to dark mode"}
            onClick={() => setDarkMode(d => !d)}
            style={{
              width: 40, height: 40, borderRadius: "50%",
              background: T.surface, border: `1px solid ${T.border}`,
              display:"flex", alignItems:"center", justifyContent:"center",
              cursor:"pointer", fontSize:16,
              backdropFilter:"blur(10px)",
              transition:"all 0.3s ease",
            }}
          >
            {darkMode ? "☀️" : "🌙"}
          </button>

          <a
            href="https://claude.ai"
            target="_blank" rel="noreferrer"
            style={{
              padding:"9px 24px",
              background: `linear-gradient(135deg, ${T.gold}, ${T.violet})`,
              borderRadius:40, color:"#fff",
              fontSize:14, fontWeight:600,
              textDecoration:"none", letterSpacing:"0.04em",
              boxShadow:`0 0 24px ${rgba(T.gold,0.35)}`,
              transition:"all 0.25s ease",
              whiteSpace:"nowrap",
            }}
            onMouseEnter={e=>{e.currentTarget.style.transform="scale(1.05)";e.currentTarget.style.boxShadow=`0 0 40px ${rgba(T.gold,0.6)}`;}}
            onMouseLeave={e=>{e.currentTarget.style.transform="scale(1)";e.currentTarget.style.boxShadow=`0 0 24px ${rgba(T.gold,0.35)}`;}}
          >
            Try Claude →
          </a>

          {/* Hamburger */}
          <button
            aria-label="Open menu"
            onClick={() => setMenuOpen(o => !o)}
            className="nav-hamburger"
            style={{
              width:40,height:40,borderRadius:8,
              background:T.surface,border:`1px solid ${T.border}`,
              display:"none",alignItems:"center",justifyContent:"center",
              cursor:"pointer",fontSize:18,color:T.text,
            }}
          >
            {menuOpen ? "✕" : "☰"}
          </button>
        </div>
      </nav>

      {/* Mobile drawer */}
      {menuOpen && (
        <div style={{
          position:"fixed",top:64,left:0,right:0,zIndex:499,
          background: darkMode ? "rgba(5,13,26,0.97)" : "rgba(247,248,252,0.97)",
          backdropFilter:"blur(24px)",
          borderBottom:`1px solid ${T.border}`,
          padding:"20px clamp(20px,4vw,56px)",
          display:"flex",flexDirection:"column",gap:4,
        }}>
          {NAV_LINKS.map(l => (
            <a key={l.label} href={l.href}
               onClick={() => setMenuOpen(false)}
               style={{ color:T.text,fontSize:17,padding:"14px 0",textDecoration:"none",fontWeight:500,borderBottom:`1px solid ${T.border}` }}>
              {l.label}
            </a>
          ))}
        </div>
      )}
    </>
  );
}

// ═══════════════════════════════════════════════════════════════════════════════
// HERO SECTION
// ═══════════════════════════════════════════════════════════════════════════════
function Hero({ T, darkMode, mouseX, mouseY, scrollRatio }) {
  const [loaded, setLoaded] = useState(false);
  useEffect(() => { const t = setTimeout(() => setLoaded(true), 100); return () => clearTimeout(t); }, []);

  const words = ["reason", "create", "write", "analyse", "build"];
  const [wordIdx, setWordIdx] = useState(0);
  const [fade, setFade] = useState(true);
  useEffect(() => {
    const iv = setInterval(() => {
      setFade(false);
      setTimeout(() => { setWordIdx(i => (i+1) % words.length); setFade(true); }, 350);
    }, 2200);
    return () => clearInterval(iv);
  }, []);

  const parallaxY = mouseY * -18;
  const parallaxX = (mouseX - 0.5) * -12;

  return (
    <section
      id="hero"
      aria-label="Hero"
      style={{
        minHeight: "100vh",
        position:"relative",
        display:"flex",flexDirection:"column",
        alignItems
        // rotate with mouse
        const cosX = Math.cos(tiltY * 0.012), sinX = Math.sin(tiltY * 0.012);
        const cosY = Math.cos(tiltX * 0.012), sinY = Math.sin(tiltX * 0.012);
        const ry = px * cosY - pz * sinY;
        const rz = px * sinY + pz * cosY;
        const ry2= py * cosX - rz * sinX;
        const rz2= py * sinX + rz * cosX;

        const scale = fov / (fov + rz2 + 200);
        return {
          sx: cx + ry * scale,
          sy: cy + ry2 * scale,
          scale,
          size: p.size * scale,
          hue: p.hue,
          alpha: clamp(scale * 1.8, 0, 1) * visible,
        };
      }).sort((a, b) => a.scale - b.scale);

      // ── connections ──
      const connV = clamp((scene - 1.2) * 2, 0, 1) * visible;
      if (connV > 0.01) {
        for (let i = 0; i < projected.length; i++) {
          for (let j = i + 1; j < projected.length; j++) {
            const dx = projected[i].sx - projected[j].sx;
            const dy = projected[i].sy - projected[j].sy;
            const d  = Math.sqrt(dx*dx + dy*dy);
            if (d < 80) {
              ctx.beginPath();
              ctx.strokeStyle = `rgba(124,92,191,${(1 - d/80) * 0.35 * connV})`;
              ctx.lineWidth = 0.5;
              ctx.moveTo(projected[i].sx, projected[i].sy);
              ctx.lineTo(projected[j].sx, projected[j].sy);
              ctx.stroke();
            }
          }
        }
      }

      // ── draw particles ──
      projected.forEach(p => {
        const grd = ctx.createRadialGradient(p.sx, p.sy, 0, p.sx, p.sy, p.size * 3);
        grd.addColorStop(0, hueColor(p.hue, p.alpha));
        grd.addColorStop(1, hueColor(p.hue, 0));
        ctx.beginPath();
        ctx.fillStyle = grd;
        ctx.arc(p.sx, p.sy, p.size * 3, 0, Math.PI * 2);
        ctx.fill();
      });

      // ── central glow ──
      const glowR = 120 + Math.sin(state.t * 1.5) * 20;
      const glow  = ctx.createRadialGradient(cx, cy, 0, cx, cy, glowR);
      glow.addColorStop(0, `rgba(193,127,62,${0.12 * visible})`);
      glow.addColorStop(0.4, `rgba(124,92,191,${0.06 * visible})`);
      glow.addColorStop(1, "transparent");
      ctx.fillStyle = glow;
      ctx.fillRect(0, 0, w, h);

      rafRef.current = requestAnimationFrame(draw);
    };

    draw();
    return () => { cancelAnimationFrame(rafRef.current); ro.disconnect(); };
  // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  // keep refs fresh without re-mounting
  useEffect(() => {}, [mouseX, mouseY, scrollY, scene]);

  return (
    <canvas
      ref={canvasRef}
      style={{
        position: "absolute", inset: 0,
        width: "100%", height: "100%",
        pointerEvents: "none",
      }}
    />
  );
}

// ─── GRID BACKGROUND ─────────────────────────────────────────────────────────
function GridBg({ opacity = 0.06 }) {
  return (
    <div style={{
      position: "absolute", inset: 0, pointerEvents: "none",
      backgroundImage: `
        linear-gradient(rgba(79,195,247,${opacity}) 1px, transparent 1px),
        linear-gradient(90deg, rgba(79,195,247,${opacity}) 1px, transparent 1px)
      `,
      backgroundSize: "60px 60px",
      maskImage: "radial-gradient(ellipse 80% 80% at 50% 50%, black 30%, transparent 100%)",
    }} />
  );
}

// ─── GLASSCARD ────────────────────────────────────────────────────────────────
function GlassCard({ children, style = {}, glow = false }) {
  return (
    <div style={{
      background: "rgba(10,22,40,0.55)",
      border: `1px solid rgba(193,127,62,${glow ? 0.5 : 0.18})`,
      borderRadius: 16,
      backdropFilter: "blur(18px)",
      boxShadow: glow
        ? "0 0 40px rgba(193,127,62,0.18), inset 0 1px 0 rgba(255,255,255,0.06)"
        : "0 8px 32px rgba(0,0,0,0.5), inset 0 1px 0 rgba(255,255,255,0.04)",
      padding: "28px 32px",
      ...style,
    }}>
      {children}
    </div>
  );
}

// ─── CAPABILITY CHIP ─────────────────────────────────────────────────────────
const CAPS = [
  { icon: "⟨/⟩", label: "Code",      color: PALETTE.ice    },
  { icon: "◈",   label: "Research",   color: PALETTE.pulse  },
  { icon: "✦",   label: "Writing",    color: PALETTE.claude },
  { icon: "◉",   label: "Analysis",   color: PALETTE.ice    },
  { icon: "⬡",   label: "Design",     color: PALETTE.pulse  },
  { icon: "◆",   label: "Education",  color: PALETTE.claude },
  { icon: "⟐",   label: "Business",   color: PALETTE.ice    },
  { icon: "✸",   label: "Creativity", color: PALETTE.pulse  },
];

function CapChip({ icon, label, color, visible, delay }) {
  const [show, setShow] = useState(false);
  useEffect(() => {
    if (!visible) { setShow(false); return; }
    const t = setTimeout(() => setShow(true), delay);
    return () => clearTimeout(t);
  }, [visible, delay]);

  return (
    <div style={{
      display: "flex", alignItems: "center", gap: 10,
      padding: "12px 20px",
      background: "rgba(10,22,40,0.7)",
      border: `1px solid ${color}44`,
      borderRadius: 40,
      backdropFilter: "blur(12px)",
      transform: show ? "translateY(0) scale(1)" : "translateY(20px) scale(0.9)",
      opacity: show ? 1 : 0,
      transition: "all 0.55s cubic-bezier(0.22,1,0.36,1)",
      cursor: "default",
      userSelect: "none",
    }}>
      <span style={{ color, fontSize: 18, fontFamily: "monospace" }}>{icon}</span>
      <span style={{ color: PALETTE.white, fontSize: 14, fontWeight: 500, letterSpacing: "0.06em" }}>{label}</span>
    </div>
  );
}

// ─── DEMO CARD ────────────────────────────────────────────────────────────────
const DEMOS = [
  {
    title: "Code Assistant",
    icon: "⟨/⟩",
    color: PALETTE.ice,
    prompt: "Write a binary search in Python",
    response: `def binary_search(arr, target):
    lo, hi = 0, len(arr) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            lo = mid + 1
        else:
            hi = mid - 1
    return -1`,
    type: "code",
  },
  {
    title: "Writing Assistant",
    icon: "✦",
    color: PALETTE.claude,
    prompt: "Open a short story about first light on Mars",
    response: `The sun rose differently here — smaller, colder, a pale coin through a rust-coloured sky. Commander Yara pressed her palm against the habitat window and watched the first true dawn on Mars paint the dunes in shades that had no names yet on Earth. She had forty-seven days of oxygen left. She smiled anyway.`,
    type: "text",
  },
  {
    title: "Research Assistant",
    icon: "◈",
    color: PALETTE.pulse,
    prompt: "Key factors behind transformer architecture success",
    response: `• Self-attention captures long-range dependencies efficiently\n• Parallel computation enables massive scale-up\n• Positional encodings preserve sequence structure\n• Layer normalisation stabilises deep training\n• Pre-training on diverse corpora transfers broadly`,
    type: "list",
  },
  {
    title: "Business Strategist",
    icon: "⟐",
    color: PALETTE.ice,
    prompt: "3-point go-to-market for a B2B SaaS product",
    response: `1. Land with a single acute pain point — resist the urge to lead with features.\n2. Instrument the activation moment ruthlessly; time-to-value is your north star metric.\n3. Design expansion loops early: usage-based pricing and team invitations compound.`,
    type: "list",
  },
  {
    title: "Design Partner",
    icon: "⬡",
    color: PALETTE.pulse,
    prompt: "Principles for a premium dark-mode UI",
    response: `Start with true black sparingly — deep navy reads more premium than #000. Reserve luminance contrast for hierarchy, not decoration. Let glows earn their place: one accent colour, one glow source. Space is your most expensive material; spend it intentionally.`,
    type: "text",
  },
];

function TypedText({ text, speed = 18, active }) {
  const [displayed, setDisplayed] = useState("");
  useEffect(() => {
    if (!active) { setDisplayed(""); return; }
    let i = 0;
    const iv = setInterval(() => {
      i++;
      setDisplayed(text.slice(0, i));
      if (i >= text.length) clearInterval(iv);
    }, speed);
    return () => clearInterval(iv);
  }, [active, text, speed]);
  return <>{displayed}<span style={{ opacity: active && displayed.length < text.length ? 1 : 0, color: PALETTE.claude }}>▋</span></>;
}

function DemoCard({ demo, active, onClick }) {
  return (
    <div
      onClick={onClick}
      style={{
        background: active ? "rgba(10,22,40,0.9)" : "rgba(5,13,26,0.6)",
        border: `1px solid ${active ? demo.color + "88" : "rgba(193,127,62,0.12)"}`,
        borderRadius: 14,
        padding: "20px 22px",
        cursor: "pointer",
        transition: "all 0.35s ease",
        boxShadow: active ? `0 0 30px ${demo.color}22` : "none",
        minWidth: 260,
        flex: "0 0 auto",
      }}
    >
      <div style={{ display: "flex", alignItems: "center", gap: 10, marginBottom: 14 }}>
        <span style={{ fontSize: 20, color: demo.color }}>{demo.icon}</span>
        <span style={{ color: PALETTE.white, fontWeight: 600, fontSize: 15 }}>{demo.title}</span>
      </div>

      {active && (
        <>
          <div style={{
            background: "rgba(0,0,0,0.4)", borderRadius: 8, padding: "10px 14px",
            marginBottom: 12, color: PALETTE.muted, fontSize: 13, fontStyle: "italic",
          }}>
            "{demo.prompt}"
          </div>
          <div style={{
            color: demo.type === "code" ? PALETTE.ice : PALETTE.white,
            fontSize: demo.type === "code" ? 12 : 13,
            fontFamily: demo.type === "code" ? "monospace" : "inherit",
            lineHeight: 1.7,
            whiteSpace: "pre-wrap",
          }}>
            <TypedText text={demo.response} speed={demo.type === "code" ? 12 : 20} active={active} />
          </div>
        </>
      )}
      {!active && (
        <div style={{ color: PALETTE.muted, fontSize: 13 }}>"{demo.prompt}"</div>
      )}
    </div>
  );
}

// ─── SECTION WRAPPER ─────────────────────────────────────────────────────────
function Section({ id, children, style = {} }) {
  return (
    <section
      id={id}
      style={{
        minHeight: "100vh",
        display: "flex",
        flexDirection: "column",
        alignItems: "center",
        justifyContent: "center",
        position: "relative",
        padding: "80px 24px",
        ...style,
      }}
    >
      {children}
    </section>
  );
}

// ─── HEADLINE ────────────────────────────────────────────────────────────────
function Headline({ text, sub, visible, size = 72, delay = 0 }) {
  const [show, setShow] = useState(false);
  useEffect(() => {
    if (!visible) { setShow(false); return; }
    const t = setTimeout(() => setShow(true), delay);
    return () => clearTimeout(t);
  }, [visible, delay]);

  const lines = text.split("\n");
  return (
    <div style={{ textAlign: "center", maxWidth: 900 }}>
      {lines.map((line, i) => (
        <div
          key={i}
          style={{
            fontSize: `clamp(${size * 0.5}px, ${size / 14}vw, ${size}px)`,
            fontWeight: 700,
            letterSpacing: "-0.02em",
            lineHeight: 1.1,
            marginBottom: 6,
            background: i % 2 === 0
              ? `linear-gradient(135deg, ${PALETTE.white} 0%, rgba(240,244,255,0.7) 100%)`
              : `linear-gradient(135deg, ${PALETTE.claude} 0%, ${PALETTE.pulse} 100%)`,
            WebkitBackgroundClip: "text",
            WebkitTextFillColor: "transparent",
            transform: show ? "translateY(0)" : "translateY(40px)",
            opacity: show ? 1 : 0,
            transition: `all 0.8s cubic-bezier(0.22,1,0.36,1) ${i * 120 + delay}ms`,
          }}
        >
          {line}
        </div>
      ))}
      {sub && (
        <div style={{
          marginTop: 24,
          color: PALETTE.muted,
          fontSize: 18,
          lineHeight: 1.6,
          transform: show ? "translateY(0)" : "translateY(20px)",
          opacity: show ? 1 : 0,
          transition: `all 0.8s cubic-bezier(0.22,1,0.36,1) ${lines.length * 120 + 100 + delay}ms`,
        }}>
          {sub}
        </div>
      )}
    </div>
  );
}

// ─── SCROLL PROGRESS DOT ─────────────────────────────────────────────────────
function ScrollDots({ current, total, onGo }) {
  return (
    <div style={{
      position: "fixed", right: 28, top: "50%",
      transform: "translateY(-50%)",
      display: "flex", flexDirection: "column", gap: 10,
      zIndex: 100,
    }}>
      {Array.from({ length: total }).map((_, i) => (
        <div
          key={i}
          onClick={() => onGo(i)}
          style={{
            width: i === current ? 10 : 6,
            height: i === current ? 10 : 6,
            borderRadius: "50%",
            background: i === current ? PALETTE.claude : "rgba(240,244,255,0.25)",
            boxShadow: i === current ? `0 0 10px ${PALETTE.claude}` : "none",
            cursor: "pointer",
            transition: "all 0.3s ease",
          }}
        />
      ))}
    </div>
  );
}

// ─── NAV ─────────────────────────────────────────────────────────────────────
function Nav({ scrolled }) {
  return (
    <nav style={{
      position: "fixed", top: 0, left: 0, right: 0,
      zIndex: 200,
      display: "flex", alignItems: "center", justifyContent: "space-between",
      padding: "0 40px",
      height: 64,
      background: scrolled ? "rgba(5,13,26,0.85)" : "transparent",
      backdropFilter: scrolled ? "blur(20px)" : "none",
      borderBottom: scrolled ? "1px solid rgba(193,127,62,0.12)" : "none",
      transition: "all 0.4s ease",
    }}>
      <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
        <div style={{
          width: 32, height: 32, borderRadius: 8,
          background: `linear-gradient(135deg, ${PALETTE.claude}, ${PALETTE.pulse})`,
          display: "flex", alignItems: "center", justifyContent: "center",
          fontSize: 16, fontWeight: 800, color: "#fff",
        }}>C</div>
        <span style={{ color: PALETTE.white, fontWeight: 600, fontSize: 16, letterSpacing: "0.02em" }}>
          Claude
        </span>
      </div>

      <div style={{ display: "flex", gap: 32 }}>
        {["Capabilities", "Demos", "About"].map(l => (
          <a key={l} href="#" style={{
            color: PALETTE.muted, fontSize: 14, textDecoration: "none",
            letterSpacing: "0.04em", fontWeight: 500,
            transition: "color 0.2s",
          }}
          onMouseEnter={e => e.target.style.color = PALETTE.white}
          onMouseLeave={e => e.target.style.color = PALETTE.muted}
          >{l}</a>
        ))}
      </div>

      <a
        href="https://claude.ai"
        target="_blank"
        rel="noreferrer"
        style={{
          padding: "9px 22px",
          background: `linear-gradient(135deg, ${PALETTE.claude}cc, ${PALETTE.pulse}cc)`,
          border: `1px solid ${PALETTE.claude}66`,
          borderRadius: 40,
          color: PALETTE.white,
          fontSize: 14, fontWeight: 600,
          textDecoration: "none",
          letterSpacing: "0.04em",
          transition: "all 0.25s ease",
          boxShadow: `0 0 20px ${PALETTE.claude}33`,
        }}
        onMouseEnter={e => { e.currentTarget.style.boxShadow = `0 0 35px ${PALETTE.claude}66`; e.currentTarget.style.transform = "scale(1.04)"; }}
        onMouseLeave={e => { e.currentTarget.style.boxShadow = `0 0 20px ${PALETTE.claude}33`; e.currentTarget.style.transform = "scale(1)"; }}
      >
        Try Claude
      </a>
    </nav>
  );
}

// ─── STAT CHIP ────────────────────────────────────────────────────────────────
function StatChip({ value, label, color, visible, delay }) {
  const [show, setShow] = useState(false);
  useEffect(() => {
    if (!visible) { setShow(false); return; }
    const t = setTimeout(() => setShow(true), delay);
    return () => clearTimeout(t);
  }, [visible, delay]);
  return (
    <div style={{
      textAlign: "center",
      transform: show ? "translateY(0)" : "translateY(30px)",
      opacity: show ? 1 : 0,
      transition: `all 0.7s cubic-bezier(0.22,1
      Continue from the previous response.

Now generate the complete core website implementation with production-ready code.

Requirements:

- Build all remaining sections
- Create advanced Three.js 3D scenes
- Implement particle systems
- Add floating holographic UI components
- Create AI-inspired animations
- Add smooth page transitions
- Implement mouse interaction effects
- Create a futuristic navigation system
- Add responsive mobile design
- Build feature showcase sections
- Create interactive statistics counters
- Implement testimonial carousel
- Add animated timeline section
- Create pricing or product comparison section
- Build a modern footer

Advanced Features:

- Dark/Light mode
- Glassmorphism effects
- Dynamic gradients
- Scroll-triggered animations
- GSAP animations
- Framer Motion animations
- Optimized performance
- Accessibility support
- SEO optimization

Code Requirements:

- Clean architecture
- Reusable components
- TypeScript support
- Tailwind CSS styling
- Modular folder structure
- Error handling
- Performance optimization

After generating the code, explain:

1. Folder structure
2. Installation process
3. Build process
4. Deployment process
5. Performance optimization techniques

Generate complete code without placeholders.

      
