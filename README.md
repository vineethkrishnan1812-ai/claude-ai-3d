import { useState, useEffect, useRef, useCallback } from "react";

// ─── CONSTANTS ────────────────────────────────────────────────────────────────
const PALETTE = {
  void:    "#020408",
  deep:    "#050d1a",
  ink:     "#0a1628",
  claude:  "#c17f3e",        // amber-gold — Claude's signature
  pulse:   "#7c5cbf",        // violet neural
  ice:     "#4fc3f7",        // data-stream cyan
  white:   "#f0f4ff",
  muted:   "rgba(240,244,255,0.45)",
};

// ─── UTILS ────────────────────────────────────────────────────────────────────
function lerp(a, b, t) { return a + (b - a) * t; }
function clamp(v, lo, hi) { return Math.max(lo, Math.min(hi, v)); }
function rand(lo, hi) { return lo + Math.random() * (hi - lo); }

// ─── PARTICLE BRAIN ───────────────────────────────────────────────────────────
function ParticleBrain({ scrollY, mouseX, mouseY, scene }) {
  const canvasRef = useRef(null);
  const stateRef  = useRef({ particles: [], connections: [], streams: [], t: 0 });
  const rafRef    = useRef(null);

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;
    const ctx = canvas.getContext("2d");
    const W = () => canvas.width  = canvas.offsetWidth  * devicePixelRatio;
    const H = () => canvas.height = canvas.offsetHeight * devicePixelRatio;
    W(); H();
    const ro = new ResizeObserver(() => { W(); H(); });
    ro.observe(canvas);

    // ── build particles ──
    const N = 200;
    const state = stateRef.current;
    state.particles = Array.from({ length: N }, (_, i) => {
      const theta = Math.random() * Math.PI * 2;
      const phi   = Math.acos(2 * Math.random() - 1);
      const r     = rand(60, 160);
      return {
        ox: Math.sin(phi) * Math.cos(theta) * r,
        oy: Math.sin(phi) * Math.sin(theta) * r,
        oz: Math.cos(phi) * r,
        x: 0, y: 0,
        size: rand(1.2, 3.5),
        speed: rand(0.3, 1.2),
        phase: rand(0, Math.PI * 2),
        hue: Math.random() < 0.6 ? "claude" : Math.random() < 0.5 ? "pulse" : "ice",
      };
    });

    // ── build streams ──
    state.streams = Array.from({ length: 30 }, () => ({
      x: rand(-canvas.width / 2, canvas.width / 2),
      y: rand(-canvas.height / 2, 0),
      len: rand(40, 120),
      speed: rand(1, 3),
      alpha: rand(0.2, 0.7),
    }));

    const hueColor = (h, a = 1) => {
      const map = { claude: PALETTE.claude, pulse: PALETTE.pulse, ice: PALETTE.ice };
      const hex = map[h];
      const r = parseInt(hex.slice(1,3),16);
      const g = parseInt(hex.slice(3,5),16);
      const b = parseInt(hex.slice(5,7),16);
      return `rgba(${r},${g},${b},${a})`;
    };

    const draw = () => {
      const w = canvas.width, h = canvas.height;
      const cx = w / 2, cy = h / 2;
      state.t += 0.008;

      ctx.clearRect(0, 0, w, h);

      // scene-driven opacity
      const visible = clamp((scene - 0.5) * 2, 0, 1) * clamp((4.5 - scene) * 1, 0, 1);
      if (visible < 0.01) { rafRef.current = requestAnimationFrame(draw); return; }

      // mouse tilt
      const tiltX = (mouseX - 0.5) * 60;
      const tiltY = (mouseY - 0.5) * 40;

      // ── data streams (scene 1+) ──
      if (scene > 0.8) {
        const sv = clamp((scene - 0.8) * 3, 0, 1);
        state.streams.forEach(s => {
          s.y += s.speed;
          if (s.y > h / 2) s.y = -h / 2;
          ctx.beginPath();
          ctx.strokeStyle = `rgba(76,195,247,${s.alpha * sv * 0.4})`;
          ctx.lineWidth = 1;
          ctx.moveTo(cx + s.x, cy + s.y);
          ctx.lineTo(cx + s.x, cy + s.y - s.len);
          ctx.stroke();
        });
      }

      // ── project & sort particles ──
      const fov = 400;
      const projected = state.particles.map(p => {
        const angle = state.t * p.speed * 0.2 + p.phase;
        const wobble = Math.sin(angle) * 8;
        const px = p.ox + wobble;
        const py = p.oy + wobble * 0.6;
        const pz = p.oz + Math.cos(angle) * 5;

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
      
