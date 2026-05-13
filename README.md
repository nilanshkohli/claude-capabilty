import { useState, useRef, useEffect } from "react";

const DS = {
  dark: "#141413", light: "#FAF9F5", midGray: "#B0AEA5",
  lightGray: "#E8E6DC", orange: "#D97757", blue: "#6A9BCC", green: "#788C5D",
  surface: "#1A1A18", border: "#252523",
};

const CAPABILITIES = {
  prompt: { label: "Simple Prompt", icon: "💬", color: DS.blue },
  skill:  { label: "Skill", icon: "⚡", color: DS.orange },
  mcp:    { label: "MCP", icon: "🔌", color: "#9B7FD4" },
  agent:  { label: "Agent", icon: "🤖", color: DS.green },
};

const CLASSIFIER_SYSTEM = `You are a capability router for Claude Assist. Given a user prompt, classify it into exactly one of: prompt, skill, mcp, agent.
- prompt: simple one-off question or single-step request
- skill: repeatable/templatable task (formatting, summarising, generating reports)
- mcp: requires external tool — calendar, email, Jira, Slack, Drive
- agent: multi-step autonomous planning or complex orchestration
Respond ONLY with valid JSON, no markdown:
{"type":"skill","reason":"One-line reason max 12 words","alternatives":{"prompt":"why not in 8 words","mcp":"why not in 8 words","agent":"why not in 8 words"}}`;

const SKILL_GUIDE_SYSTEM = `You are Claude Assist, a capability guidance engine. Given a user's prompt, generate a structured Skill guide.
Respond ONLY with valid JSON (no markdown, no backticks):
{
  "skillName": "Short descriptive skill name (4-6 words)",
  "overview": { "what": "What this skill does in 1-2 sentences", "problem": "Problem it solves in 1-2 sentences" },
  "whenToUse": ["use case 1", "use case 2", "use case 3"],
  "whenNotToUse": ["scenario 1", "scenario 2"],
  "betterAlternative": "What to use instead and why",
  "steps": [
    {"title":"Preparation","desc":"..."},
    {"title":"How to start","desc":"..."},
    {"title":"Configuration","desc":"..."},
    {"title":"Execution","desc":"..."},
    {"title":"Output understanding","desc":"..."},
    {"title":"Iteration","desc":"..."}
  ],
  "example": { "input": "Example input prompt", "output": "Example output result" },
  "proTips": ["tip 1", "tip 2", "tip 3"],
  "cost": { "tokens": "Estimated token usage", "latency": "Expected response time", "limits": "Any known limits" },
  "combinations": { "withAgents": "How to combine with Agents", "withWorkflows": "How to combine with Workflows" },
  "available": true
}`;



const SectionCard = ({ title, accent, children }) => (
  <div style={{ background: DS.surface, borderRadius: 12, border: `1px solid ${DS.border}`, borderLeft: `4px solid ${accent || DS.orange}`, marginBottom: 14, overflow: "hidden" }}>
    <div style={{ background: DS.orange, padding: "5px 14px", display: "inline-block" }}>
      <span style={{ color: DS.dark, fontSize: 10, fontWeight: 700, letterSpacing: 1.5, textTransform: "uppercase", fontFamily: "Poppins, Arial" }}>{title}</span>
    </div>
    <div style={{ padding: "14px 18px" }}>{children}</div>
  </div>
);

const Bullet = ({ children, color }) => (
  <div style={{ display: "flex", gap: 10, marginBottom: 8, alignItems: "flex-start" }}>
    <div style={{ width: 6, height: 6, borderRadius: "50%", background: color || DS.orange, marginTop: 6, flexShrink: 0 }} />
    <p style={{ color: DS.midGray, fontSize: 13, margin: 0, lineHeight: 1.6 }}>{children}</p>
  </div>
);

const SkeletonLine = ({ w }) => (
  <div style={{ height: 12, borderRadius: 6, background: "#252523", width: w || "100%", marginBottom: 8, animation: "shimmer 1.5s ease-in-out infinite" }} />
);

export default function App() {
  const [screen, setScreen] = useState("onboard"); // onboard | chat | skill-guide | coming-soon | dashboard
  const [onboardStep, setOnboardStep] = useState(0);
  const [onboardData, setOnboardData] = useState({ uses: [], frequency: "" });
  const [inputVal, setInputVal] = useState("");
  const [hasTyped, setHasTyped] = useState(false);
  const [messages, setMessages] = useState([
    { role: "claude", text: "Hi! I'm Claude Assist — I'll recommend the right capability (Skill, Agent, MCP, or Prompt) for every task you send me. Skill guides are fully available now. Just type what you want to do. ✦" }
  ]);
  const [panel, setPanel] = useState(null);
  const [panelVisible, setPanelVisible] = useState(false);
  const [classifying, setClassifying] = useState(false);
  const [skillGuide, setSkillGuide] = useState(null);
  const [skillLoading, setSkillLoading] = useState(false);
  const [comingSoonCap, setComingSoonCap] = useState(null);
  const [pendingPrompt, setPendingPrompt] = useState("");
  const [skills, setSkills] = useState([
    { id: 1, name: "Sprint Summary Generator", desc: "Formats sprint data into stakeholder summaries", savings: 14800, uses: 12, created: "Apr 20" },
    { id: 2, name: "Access Review Emailer", desc: "Composes UAR notification emails from template", savings: 9200, uses: 7, created: "Apr 24" },
  ]);
  const [stats, setStats] = useState({ prompt: 0, skill: 0, mcp: 0, agent: 0 });
  const chatEndRef = useRef(null);

  useEffect(() => { chatEndRef.current?.scrollIntoView({ behavior: "smooth" }); }, [messages]);

  const callClaude = async (system, userMsg, maxTokens = 300) => {
    const res = await fetch("https://api.anthropic.com/v1/messages", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        model: "claude-sonnet-4-20250514",
        max_tokens: maxTokens,
        system,
        messages: [{ role: "user", content: userMsg }],
      }),
    });
    const data = await res.json();
    const raw = data.content?.find(b => b.type === "text")?.text || "{}";
    return JSON.parse(raw.replace(/```json|```/g, "").trim());
  };

  const handleSend = async () => {
    const text = inputVal.trim();
    if (!text || classifying) return;
    setInputVal("");
    setHasTyped(false);
    setPendingPrompt(text);
    setClassifying(true);
    try {
      const result = await callClaude(CLASSIFIER_SYSTEM, text, 200);
      setPanel({ ...result, prompt: text });
      setPanelVisible(true);
    } catch {
      setPanel({ type: "prompt", reason: "Defaulting to simple prompt", alternatives: { skill: "may apply if repeated", mcp: "no tool needed", agent: "no multi-step needed" }, prompt: text });
      setPanelVisible(true);
    } finally {
      setClassifying(false);
    }
  };

  const handleCapabilityClick = async (type, prompt) => {
    setPanelVisible(false);
    setStats(s => ({ ...s, [type]: s[type] + 1 }));
    if (type === "skill") {
      setSkillLoading(true);
      setScreen("skill-guide");
      setSkillGuide(null);
      try {
        const guide = await callClaude(SKILL_GUIDE_SYSTEM, prompt, 1000);
        setSkillGuide(guide);
      } catch {
        setSkillGuide({ skillName: "Custom Skill", overview: { what: "Could not generate guide.", problem: "Please try again." }, whenToUse: [], whenNotToUse: [], betterAlternative: "", steps: [], example: { input: "", output: "" }, proTips: [], cost: { tokens: "—", latency: "—", limits: "—" }, combinations: { withAgents: "—", withWorkflows: "—" }, available: true });
      } finally {
        setSkillLoading(false);
      }
    } else {
      setComingSoonCap(type);
      setScreen("coming-soon");
    }
  };

  const startSkill = () => {
    if (!skillGuide) return;
    setSkills(s => [...s, { id: Date.now(), name: skillGuide.skillName, desc: skillGuide.overview.what, savings: Math.floor(Math.random() * 2000) + 800, uses: 0, created: "May 2" }]);
    setMessages(m => [...m,
      { role: "user", text: pendingPrompt, capType: "skill" },
      { role: "claude", text: `✅ Skill "${skillGuide.skillName}" is now active. I'll use this template for this and future similar requests.` }
    ]);
    setScreen("chat");
    setSkillGuide(null);
  };

  const fmtTokens = n => n >= 1000 ? `${(n / 1000).toFixed(1)}k` : n;
  const totalSavings = skills.reduce((a, s) => a + s.savings, 0);



  // ── SKILL GUIDE ───────────────────────────────────────────────────────────
  if (screen === "skill-guide") return (
    <div style={{ background: DS.dark, minHeight: "100vh", fontFamily: "Poppins, Arial, sans-serif", display: "flex", flexDirection: "column" }}>
      <div style={{ borderBottom: `1px solid ${DS.border}`, padding: "0 20px", height: 52, display: "flex", alignItems: "center", justifyContent: "space-between", flexShrink: 0 }}>
        <button onClick={() => { setScreen("chat"); setSkillGuide(null); }} style={{ background: "transparent", border: "none", color: DS.midGray, cursor: "pointer", fontFamily: "Poppins, Arial", fontSize: 13, display: "flex", alignItems: "center", gap: 6 }}>
          ← Back
        </button>
        <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
          <span style={{ fontSize: 16 }}>⚡</span>
          <span style={{ color: DS.light, fontWeight: 700, fontSize: 14 }}>{skillLoading ? "Generating Skill Guide…" : (skillGuide?.skillName || "Skill Guide")}</span>
        </div>
        <button onClick={startSkill} disabled={skillLoading || !skillGuide}
          style={{ padding: "8px 18px", borderRadius: 8, background: skillLoading || !skillGuide ? "#2A2A28" : DS.orange, color: skillLoading || !skillGuide ? DS.midGray : DS.dark, fontFamily: "Poppins, Arial", fontWeight: 700, fontSize: 13, border: "none", cursor: skillLoading || !skillGuide ? "default" : "pointer" }}>
          Start Skill →
        </button>
      </div>

      <div style={{ flex: 1, overflowY: "auto", maxWidth: 720, margin: "0 auto", width: "100%", padding: "24px 16px 40px" }}>
        {skillLoading ? (
          <div>
            <div style={{ background: `${DS.orange}18`, border: `1px solid ${DS.orange}33`, borderRadius: 10, padding: "14px 18px", marginBottom: 20, display: "flex", alignItems: "center", gap: 12 }}>
              <div style={{ width: 8, height: 8, borderRadius: "50%", background: DS.orange, animation: "pulse 1s ease-in-out infinite" }} />
              <p style={{ color: DS.midGray, fontSize: 13, margin: 0 }}>Claude is analysing your prompt and generating a tailored Skill guide…</p>
            </div>
            {["80%","60%","90%","50%","70%","40%"].map((w, i) => <SkeletonLine key={i} w={w} />)}
          </div>
        ) : skillGuide ? (
          <>
            {/* Overview */}
            <SectionCard title="Skill Overview">
              <p style={{ color: DS.midGray, fontSize: 12, fontWeight: 700, margin: "0 0 4px", letterSpacing: 0.5 }}>WHAT THIS SKILL DOES</p>
              <p style={{ color: DS.light, fontSize: 14, margin: "0 0 14px", lineHeight: 1.6 }}>{skillGuide.overview.what}</p>
              <p style={{ color: DS.midGray, fontSize: 12, fontWeight: 700, margin: "0 0 4px", letterSpacing: 0.5 }}>PROBLEM IT SOLVES</p>
              <p style={{ color: DS.light, fontSize: 14, margin: 0, lineHeight: 1.6 }}>{skillGuide.overview.problem}</p>
            </SectionCard>

            {/* When to use */}
            <SectionCard title="When to Use" accent={DS.green}>
              {skillGuide.whenToUse.map((u, i) => <Bullet key={i} color={DS.green}>Use case {i + 1}: {u}</Bullet>)}
            </SectionCard>

            {/* When not to use */}
            <SectionCard title="When Not to Use" accent="#E05C5C">
              {skillGuide.whenNotToUse.map((u, i) => <Bullet key={i} color="#E05C5C">{u}</Bullet>)}
              <div style={{ background: "#1E1E1C", borderRadius: 8, padding: "10px 14px", marginTop: 10 }}>
                <p style={{ color: DS.midGray, fontSize: 11, fontWeight: 700, letterSpacing: 0.5, margin: "0 0 4px" }}>BETTER ALTERNATIVE</p>
                <p style={{ color: DS.light, fontSize: 13, margin: 0 }}>{skillGuide.betterAlternative}</p>
              </div>
            </SectionCard>

            {/* Steps */}
            <SectionCard title="Step-by-Step Guide">
              {skillGuide.steps.map((s, i) => (
                <div key={i} style={{ display: "flex", gap: 14, marginBottom: 14, alignItems: "flex-start" }}>
                  <div style={{ width: 28, height: 28, borderRadius: 8, background: `${DS.orange}22`, border: `1px solid ${DS.orange}44`, display: "flex", alignItems: "center", justifyContent: "center", flexShrink: 0 }}>
                    <span style={{ color: DS.orange, fontSize: 12, fontWeight: 700 }}>{i + 1}</span>
                  </div>
                  <div>
                    <p style={{ color: DS.light, fontSize: 13, fontWeight: 700, margin: "0 0 3px" }}>{s.title}</p>
                    <p style={{ color: DS.midGray, fontSize: 13, margin: 0, lineHeight: 1.6 }}>{s.desc}</p>
                  </div>
                </div>
              ))}
            </SectionCard>

            {/* Example */}
            <SectionCard title="Example" accent={DS.blue}>
              <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: 12 }}>
                {[["INPUT", skillGuide.example.input, "#2A2A28"], ["OUTPUT", skillGuide.example.output, `${DS.blue}11`]].map(([lbl, val, bg]) => (
                  <div key={lbl} style={{ background: bg, borderRadius: 8, padding: "12px 14px" }}>
                    <p style={{ color: DS.midGray, fontSize: 10, fontWeight: 700, letterSpacing: 1, margin: "0 0 6px" }}>{lbl}</p>
                    <p style={{ color: DS.light, fontSize: 12, margin: 0, lineHeight: 1.6, fontStyle: "italic" }}>{val}</p>
                  </div>
                ))}
              </div>
            </SectionCard>

            {/* Pro Tips */}
            <SectionCard title="Pro Tips" accent={DS.orange}>
              {skillGuide.proTips.map((t, i) => <Bullet key={i}>Tip {i + 1}: {t}</Bullet>)}
            </SectionCard>

            {/* Cost */}
            <SectionCard title="Cost & Performance" accent="#9B7FD4">
              <div style={{ display: "grid", gridTemplateColumns: "repeat(3,1fr)", gap: 10 }}>
                {[["Token Usage", skillGuide.cost.tokens], ["Latency", skillGuide.cost.latency], ["Limits", skillGuide.cost.limits]].map(([lbl, val]) => (
                  <div key={lbl} style={{ background: "#1E1E1C", borderRadius: 8, padding: "10px 12px" }}>
                    <p style={{ color: DS.midGray, fontSize: 10, fontWeight: 700, letterSpacing: 0.5, margin: "0 0 6px" }}>{lbl.toUpperCase()}</p>
                    <p style={{ color: DS.light, fontSize: 12, margin: 0, lineHeight: 1.5 }}>{val}</p>
                  </div>
                ))}
              </div>
            </SectionCard>

            {/* Combinations */}
            <SectionCard title="Combination Usage" accent={DS.green}>
              <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: 10 }}>
                {[["🤖 With Agents", skillGuide.combinations.withAgents], ["🔄 With Workflows", skillGuide.combinations.withWorkflows]].map(([lbl, val]) => (
                  <div key={lbl} style={{ background: "#1E1E1C", borderRadius: 8, padding: "12px 14px" }}>
                    <p style={{ color: DS.green, fontSize: 12, fontWeight: 700, margin: "0 0 6px" }}>{lbl}</p>
                    <p style={{ color: DS.midGray, fontSize: 12, margin: 0, lineHeight: 1.5 }}>{val}</p>
                  </div>
                ))}
              </div>
            </SectionCard>

            {/* Availability */}
            <SectionCard title="Availability" accent={DS.orange}>
              <div style={{ display: "flex", alignItems: "center", gap: 12 }}>
                <div style={{ width: 36, height: 36, borderRadius: 10, background: `${DS.green}22`, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 18, flexShrink: 0 }}>✓</div>
                <div>
                  <p style={{ color: DS.green, fontWeight: 700, fontSize: 14, margin: "0 0 3px" }}>Available Now</p>
                  <p style={{ color: DS.midGray, fontSize: 13, margin: 0 }}>This Skill is ready to use. Click "Start Skill →" to activate it.</p>
                </div>
              </div>
            </SectionCard>

            <button onClick={startSkill} style={{ width: "100%", padding: 16, borderRadius: 10, background: DS.orange, color: DS.dark, fontFamily: "Poppins, Arial", fontWeight: 700, fontSize: 15, border: "none", cursor: "pointer", marginTop: 8 }}>
              ⚡ Start Skill — {skillGuide.skillName}
            </button>
          </>
        ) : null}
      </div>
    </div>
  );

  // ── COMING SOON ───────────────────────────────────────────────────────────
  if (screen === "coming-soon") {
    const cap = CAPABILITIES[comingSoonCap] || {};
    const V2_INFO = {
      mcp: { title: "MCP (Model Context Protocol)", desc: "MCP lets Claude connect to external tools — your calendar, email, Jira, Slack, Google Drive, and more. Instead of copy-pasting data into Claude, MCP pulls it directly.", why: "We're building secure OAuth flows and tool-selection UX. Coming in v2.", eta: "Q3 2025" },
      agent: { title: "Agent Mode", desc: "Agents let Claude autonomously plan and execute multi-step tasks — researching, writing, coding, and iterating without you having to prompt each step.", why: "We're designing human-in-the-loop checkpoints and cost guardrails first. Coming in v2.", eta: "Q4 2025" },
      prompt: { title: "Simple Prompt", desc: "Just type and get a response — no setup, no templates. Best for one-off questions, explanations, and quick tasks.", why: "This already works in Chat. Head back and try it!", eta: "Available now" },
    };
    const info = V2_INFO[comingSoonCap] || {};
    return (
      <div style={{ background: DS.dark, minHeight: "100vh", fontFamily: "Poppins, Arial, sans-serif", display: "flex", flexDirection: "column" }}>
        <div style={{ borderBottom: `1px solid ${DS.border}`, padding: "0 20px", height: 52, display: "flex", alignItems: "center" }}>
          <button onClick={() => setScreen("chat")} style={{ background: "transparent", border: "none", color: DS.midGray, cursor: "pointer", fontFamily: "Poppins, Arial", fontSize: 13, display: "flex", alignItems: "center", gap: 6 }}>← Back to Chat</button>
        </div>
        <div style={{ flex: 1, display: "flex", flexDirection: "column", alignItems: "center", justifyContent: "center", padding: "2rem", textAlign: "center" }}>
          <div style={{ width: 64, height: 64, borderRadius: 16, background: `${cap.color}18`, border: `1px solid ${cap.color}44`, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 28, marginBottom: 20 }}>{cap.icon}</div>
          <span style={{ background: `${cap.color}22`, color: cap.color, fontSize: 10, fontWeight: 700, padding: "3px 10px", borderRadius: 4, letterSpacing: 1.5, marginBottom: 16, display: "inline-block" }}>
            {comingSoonCap === "prompt" ? "AVAILABLE NOW" : "COMING IN V2"}
          </span>
          <h2 style={{ color: DS.light, fontSize: 24, fontWeight: 700, marginBottom: 12 }}>{info.title}</h2>
          <p style={{ color: DS.midGray, fontSize: 15, maxWidth: 440, lineHeight: 1.7, marginBottom: 20 }}>{info.desc}</p>
          <div style={{ background: DS.surface, border: `1px solid ${DS.border}`, borderRadius: 10, padding: "14px 20px", maxWidth: 440, width: "100%", marginBottom: 28, textAlign: "left" }}>
            <p style={{ color: DS.midGray, fontSize: 12, fontWeight: 700, letterSpacing: 0.5, margin: "0 0 6px" }}>WHY NOT IN MVP</p>
            <p style={{ color: DS.light, fontSize: 13, margin: "0 0 12px", lineHeight: 1.6 }}>{info.why}</p>
            <p style={{ color: DS.midGray, fontSize: 12, fontWeight: 700, letterSpacing: 0.5, margin: "0 0 4px" }}>ETA</p>
            <p style={{ color: cap.color, fontSize: 13, fontWeight: 700, margin: 0 }}>{info.eta}</p>
          </div>
          <button onClick={() => setScreen("chat")} style={{ padding: "12px 28px", borderRadius: 8, background: DS.orange, color: DS.dark, fontFamily: "Poppins, Arial", fontWeight: 700, fontSize: 14, border: "none", cursor: "pointer" }}>
            ⚡ Use Skill instead — it's ready now
          </button>
        </div>
      </div>
    );
  }

  // ── CHAT ──────────────────────────────────────────────────────────────────
  if (screen === "chat") return (
    <div style={{ background: DS.dark, minHeight: "100vh", fontFamily: "Poppins, Arial, sans-serif", display: "flex", flexDirection: "column" }}>
      <div style={{ borderBottom: `1px solid ${DS.border}`, padding: "0 24px", height: 52, display: "flex", alignItems: "center", justifyContent: "space-between", flexShrink: 0 }}>
        <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
          <div style={{ width: 28, height: 28, borderRadius: 6, background: DS.orange, display: "flex", alignItems: "center", justifyContent: "center" }}>
            <span style={{ color: DS.dark, fontSize: 14, fontWeight: 700 }}>A</span>
          </div>
          <span style={{ color: DS.light, fontWeight: 700, fontSize: 15 }}>Claude Assist</span>
          <span style={{ background: `${DS.orange}22`, color: DS.orange, fontSize: 10, fontWeight: 700, padding: "2px 8px", borderRadius: 4, letterSpacing: 1 }}>MVP</span>
        </div>
        <div style={{ display: "flex", gap: 4 }}>
          {["chat", "dashboard"].map(s => (
            <button key={s} onClick={() => setScreen(s)} style={{ padding: "6px 14px", borderRadius: 6, border: "none", background: screen === s ? "#222220" : "transparent", color: screen === s ? DS.light : DS.midGray, fontFamily: "Poppins, Arial", fontWeight: 600, fontSize: 12, cursor: "pointer", textTransform: "capitalize" }}>{s === "chat" ? "Chat" : "Dashboard"}</button>
          ))}
        </div>
      </div>

      <div style={{ flex: 1, display: "flex", flexDirection: "column", maxWidth: 720, margin: "0 auto", width: "100%", padding: "0 16px" }}>
        <div style={{ flex: 1, overflowY: "auto", padding: "24px 0 8px" }}>
          {messages.length === 0 && !classifying && !hasTyped && (
            <div style={{ textAlign: "center", padding: "80px 0 32px" }}>
              <div style={{ width: 52, height: 52, borderRadius: 14, background: `${DS.orange}18`, border: `1px solid ${DS.orange}44`, display: "flex", alignItems: "center", justifyContent: "center", margin: "0 auto 16px", fontSize: 24 }}>✦</div>
              <p style={{ color: DS.light, fontSize: 18, fontWeight: 600, marginBottom: 8 }}>What would you like to do?</p>
              <p style={{ color: DS.midGray, fontSize: 14, lineHeight: 1.6 }}>Type a prompt. Claude Assist will recommend<br />the right capability before executing.</p>
            </div>
          )}
          {messages.map((m, i) => (
            <div key={i} style={{ display: "flex", justifyContent: m.role === "user" ? "flex-end" : "flex-start", marginBottom: 12, alignItems: "flex-end", gap: 8 }}>
              {m.role === "claude" && <div style={{ width: 28, height: 28, borderRadius: 8, background: DS.orange, display: "flex", alignItems: "center", justifyContent: "center", flexShrink: 0, marginBottom: 2 }}><span style={{ color: DS.dark, fontSize: 12, fontWeight: 700 }}>A</span></div>}
              <div style={{ maxWidth: "78%" }}>
                {m.role === "user" && m.capType && (
                  <div style={{ display: "flex", justifyContent: "flex-end", marginBottom: 4 }}>
                    <span style={{ fontSize: 10, fontWeight: 700, padding: "2px 8px", borderRadius: 4, background: `${CAPABILITIES[m.capType].color}22`, color: CAPABILITIES[m.capType].color, letterSpacing: 0.5 }}>{CAPABILITIES[m.capType].icon} {CAPABILITIES[m.capType].label.toUpperCase()}</span>
                  </div>
                )}
                <div style={{ background: m.role === "user" ? DS.orange : DS.surface, borderRadius: m.role === "user" ? "16px 16px 4px 16px" : "16px 16px 16px 4px", padding: "10px 14px", border: m.role === "claude" ? `1px solid ${DS.border}` : "none" }}>
                  <p style={{ color: m.role === "user" ? DS.dark : DS.light, fontSize: 14, margin: 0, lineHeight: 1.6 }}>{m.text}</p>
                </div>
              </div>
            </div>
          ))}
          {classifying && (
            <div style={{ display: "flex", alignItems: "center", gap: 10, padding: "12px 0" }}>
              <div style={{ width: 28, height: 28, borderRadius: 8, background: DS.orange, display: "flex", alignItems: "center", justifyContent: "center" }}><span style={{ color: DS.dark, fontSize: 12, fontWeight: 700 }}>A</span></div>
              <div style={{ display: "flex", gap: 5 }}>
                {[0, 1, 2].map(i => <div key={i} style={{ width: 6, height: 6, borderRadius: "50%", background: DS.orange, opacity: 0.6 }} />)}
              </div>
              <span style={{ color: DS.midGray, fontSize: 12 }}>Routing your prompt…</span>
            </div>
          )}
          <div ref={chatEndRef} />
        </div>

        {/* Capability Panel */}
        {panelVisible && panel && (
          <div style={{ background: DS.surface, border: `1px solid ${DS.border}`, borderRadius: "16px 16px 0 0", padding: "20px 20px 24px", borderBottom: "none" }}>
            <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 16 }}>
              <div>
                <p style={{ fontSize: 11, fontWeight: 700, letterSpacing: 1.5, color: DS.orange, textTransform: "uppercase", margin: "0 0 4px" }}>Claude Assist recommends</p>
                <p style={{ color: DS.midGray, fontSize: 12, margin: 0, fontStyle: "italic" }}>"{panel.prompt?.slice(0, 60)}{panel.prompt?.length > 60 ? "…" : ""}"</p>
              </div>
              <button onClick={() => { setPanelVisible(false); setMessages(m => [...m, { role: "user", text: panel.prompt }, { role: "claude", text: "Got it — responding as a simple prompt." }]); }}
                style={{ background: "transparent", border: `1px solid ${DS.border}`, color: DS.midGray, borderRadius: 6, padding: "5px 12px", cursor: "pointer", fontFamily: "Poppins, Arial", fontSize: 11, fontWeight: 600 }}>Just prompt</button>
            </div>
            <div style={{ background: `${CAPABILITIES[panel.type]?.color}12`, border: `1.5px solid ${CAPABILITIES[panel.type]?.color}55`, borderRadius: 12, padding: "14px 16px", marginBottom: 12, display: "flex", alignItems: "center", gap: 14 }}>
              <div style={{ width: 44, height: 44, borderRadius: 10, background: `${CAPABILITIES[panel.type]?.color}22`, display: "flex", alignItems: "center", justifyContent: "center", flexShrink: 0, fontSize: 22 }}>{CAPABILITIES[panel.type]?.icon}</div>
              <div style={{ flex: 1 }}>
                <div style={{ display: "flex", alignItems: "center", gap: 8, marginBottom: 4 }}>
                  <p style={{ color: CAPABILITIES[panel.type]?.color, fontWeight: 700, fontSize: 15, margin: 0 }}>{CAPABILITIES[panel.type]?.label}</p>
                  <span style={{ background: `${CAPABILITIES[panel.type]?.color}22`, color: CAPABILITIES[panel.type]?.color, fontSize: 10, fontWeight: 700, padding: "1px 6px", borderRadius: 4, letterSpacing: 0.5 }}>BEST FIT</span>
                  {panel.type === "skill" && <span style={{ background: `${DS.green}22`, color: DS.green, fontSize: 10, fontWeight: 700, padding: "1px 6px", borderRadius: 4 }}>FULL GUIDE AVAILABLE</span>}
                </div>
                <p style={{ color: DS.midGray, fontSize: 12, margin: "0 0 10px", lineHeight: 1.5 }}>{panel.reason}</p>
                <button onClick={() => handleCapabilityClick(panel.type, panel.prompt)}
                  style={{ padding: "8px 20px", borderRadius: 7, background: CAPABILITIES[panel.type]?.color, color: DS.dark, fontFamily: "Poppins, Arial", fontWeight: 700, fontSize: 13, border: "none", cursor: "pointer" }}>
                  {panel.type === "skill" ? "⚡ View Skill Guide →" : `Use ${CAPABILITIES[panel.type]?.label} →`}
                </button>
              </div>
            </div>
            <div style={{ display: "grid", gridTemplateColumns: "repeat(3, 1fr)", gap: 8 }}>
              {Object.entries(panel.alternatives || {}).map(([type, reason]) => {
                const cap = CAPABILITIES[type]; if (!cap) return null;
                return (
                  <button key={type} onClick={() => handleCapabilityClick(type, panel.prompt)}
                    style={{ background: "transparent", border: `1px solid ${DS.border}`, borderRadius: 10, padding: "10px 12px", cursor: "pointer", textAlign: "left" }}
                    onMouseEnter={e => e.currentTarget.style.borderColor = cap.color}
                    onMouseLeave={e => e.currentTarget.style.borderColor = DS.border}>
                    <div style={{ display: "flex", alignItems: "center", gap: 6, marginBottom: 4 }}>
                      <span style={{ fontSize: 14 }}>{cap.icon}</span>
                      <span style={{ color: cap.color, fontWeight: 700, fontSize: 12 }}>{cap.label}</span>
                    </div>
                    <p style={{ color: "#555553", fontSize: 11, margin: 0, lineHeight: 1.4 }}>{reason}</p>
                  </button>
                );
              })}
            </div>
          </div>
        )}

        {/* Input */}
        <div style={{ background: panelVisible ? DS.surface : "transparent", borderTop: panelVisible ? `1px solid ${DS.border}` : "none", padding: "12px 0 20px", flexShrink: 0 }}>
          <div style={{ background: "#1A1A18", border: `1px solid ${DS.border}`, borderRadius: 12, display: "flex", alignItems: "flex-end", gap: 8, padding: "10px 12px" }}>
            <textarea value={inputVal}
              onChange={e => { setInputVal(e.target.value); setHasTyped(e.target.value.length > 0); }}
              onKeyDown={e => { if (e.key === "Enter" && !e.shiftKey && !classifying) { e.preventDefault(); handleSend(); } }}
              placeholder="Ask Claude anything…" rows={1}
              style={{ flex: 1, background: "transparent", border: "none", outline: "none", color: DS.light, fontFamily: "Poppins, Arial", fontSize: 14, resize: "none", lineHeight: 1.6, maxHeight: 120, overflowY: "auto" }} />
            <button onClick={handleSend} disabled={!inputVal.trim() || classifying}
              style={{ width: 34, height: 34, borderRadius: 8, background: inputVal.trim() && !classifying ? DS.orange : "#2A2A28", border: "none", cursor: inputVal.trim() && !classifying ? "pointer" : "default", display: "flex", alignItems: "center", justifyContent: "center", flexShrink: 0, transition: "background 0.2s" }}>
              {classifying
                ? <div style={{ width: 14, height: 14, border: `2px solid ${DS.midGray}`, borderTopColor: "transparent", borderRadius: "50%", animation: "spin 0.8s linear infinite" }} />
                : <svg width="14" height="14" viewBox="0 0 24 24" fill="none"><path d="M22 2L11 13" stroke={inputVal.trim() ? DS.dark : DS.midGray} strokeWidth="2" strokeLinecap="round" /><path d="M22 2L15 22L11 13L2 9L22 2Z" stroke={inputVal.trim() ? DS.dark : DS.midGray} strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" /></svg>
              }
            </button>
          </div>
          <p style={{ color: "#3A3A38", fontSize: 11, textAlign: "center", marginTop: 8 }}>Claude Assist routes every prompt to the right capability</p>
        </div>
      </div>
    </div>
  );

  // ── DASHBOARD ─────────────────────────────────────────────────────────────
  return (
    <div style={{ background: DS.dark, minHeight: "100vh", fontFamily: "Poppins, Arial, sans-serif", display: "flex", flexDirection: "column" }}>
      <div style={{ borderBottom: `1px solid ${DS.border}`, padding: "0 24px", height: 52, display: "flex", alignItems: "center", justifyContent: "space-between", flexShrink: 0 }}>
        <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
          <div style={{ width: 28, height: 28, borderRadius: 6, background: DS.orange, display: "flex", alignItems: "center", justifyContent: "center" }}><span style={{ color: DS.dark, fontSize: 14, fontWeight: 700 }}>A</span></div>
          <span style={{ color: DS.light, fontWeight: 700, fontSize: 15 }}>Claude Assist</span>
        </div>
        <div style={{ display: "flex", gap: 4 }}>
          {["chat", "dashboard"].map(s => (
            <button key={s} onClick={() => setScreen(s)} style={{ padding: "6px 14px", borderRadius: 6, border: "none", background: screen === s ? "#222220" : "transparent", color: screen === s ? DS.light : DS.midGray, fontFamily: "Poppins, Arial", fontWeight: 600, fontSize: 12, cursor: "pointer", textTransform: "capitalize" }}>{s === "chat" ? "Chat" : "Dashboard"}</button>
          ))}
        </div>
      </div>
      <div style={{ flex: 1, overflowY: "auto", maxWidth: 800, margin: "0 auto", width: "100%", padding: "24px 16px" }}>
        <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 24 }}>
          <div>
            <h2 style={{ color: DS.light, fontSize: 22, fontWeight: 700, margin: "0 0 4px" }}>Dashboard</h2>
            <p style={{ color: DS.midGray, fontSize: 13, margin: 0 }}>Skill adoption metrics · MVP focus</p>
          </div>
          <button onClick={() => setScreen("chat")} style={{ padding: "9px 18px", borderRadius: 8, background: DS.orange, color: DS.dark, fontFamily: "Poppins, Arial", fontWeight: 700, fontSize: 13, border: "none", cursor: "pointer" }}>+ New prompt</button>
        </div>
        <p style={{ color: DS.midGray, fontSize: 11, fontWeight: 700, letterSpacing: 1.5, textTransform: "uppercase", marginBottom: 12 }}>Capability routing this session</p>
        <div style={{ display: "grid", gridTemplateColumns: "repeat(4, 1fr)", gap: 10, marginBottom: 28 }}>
          {Object.entries(CAPABILITIES).map(([k, v]) => (
            <div key={k} style={{ background: DS.surface, borderRadius: 10, padding: "14px", border: `1px solid ${k === "skill" ? DS.orange + "55" : DS.border}` }}>
              <div style={{ display: "flex", alignItems: "center", gap: 6, marginBottom: 10 }}>
                <span style={{ fontSize: 16 }}>{v.icon}</span>
                <span style={{ color: v.color, fontSize: 11, fontWeight: 700 }}>{v.label}</span>
              </div>
              <p style={{ color: DS.orange, fontSize: 26, fontWeight: 700, margin: "0 0 2px", fontFamily: "Poppins, Arial" }}>{stats[k]}</p>
              <p style={{ color: "#555553", fontSize: 10, margin: 0 }}>{k === "skill" ? "✓ MVP" : k === "prompt" ? "available" : "v2"}</p>
            </div>
          ))}
        </div>
        <p style={{ color: DS.midGray, fontSize: 11, fontWeight: 700, letterSpacing: 1.5, textTransform: "uppercase", marginBottom: 12 }}>Saved Skills</p>
        <div style={{ display: "grid", gridTemplateColumns: "repeat(3, 1fr)", gap: 10, marginBottom: 20 }}>
          {[{ label: "Skills created", val: skills.length }, { label: "Tokens saved", val: fmtTokens(totalSavings) }, { label: "Total uses", val: skills.reduce((a, s) => a + s.uses, 0) }].map(s => (
            <div key={s.label} style={{ background: DS.surface, borderRadius: 10, padding: "16px", border: `1px solid ${DS.border}` }}>
              <p style={{ color: DS.midGray, fontSize: 11, fontWeight: 600, letterSpacing: 0.5, textTransform: "uppercase", margin: "0 0 8px" }}>{s.label}</p>
              <p style={{ color: DS.orange, fontSize: 26, fontWeight: 700, margin: 0, fontFamily: "Poppins, Arial" }}>{s.val}</p>
            </div>
          ))}
        </div>
        <div style={{ display: "flex", flexDirection: "column", gap: 8 }}>
          {skills.map(sk => (
            <div key={sk.id} style={{ background: DS.surface, borderRadius: 10, padding: "14px 18px", border: `1px solid ${DS.border}`, display: "flex", alignItems: "center", gap: 14 }}>
              <div style={{ width: 38, height: 38, borderRadius: 9, background: `${DS.orange}18`, border: `1px solid ${DS.orange}33`, display: "flex", alignItems: "center", justifyContent: "center", flexShrink: 0, fontSize: 16 }}>⚡</div>
              <div style={{ flex: 1, minWidth: 0 }}>
                <p style={{ color: DS.light, fontWeight: 600, fontSize: 14, margin: "0 0 3px", overflow: "hidden", textOverflow: "ellipsis", whiteSpace: "nowrap" }}>{sk.name}</p>
                <p style={{ color: DS.midGray, fontSize: 12, margin: 0, overflow: "hidden", textOverflow: "ellipsis", whiteSpace: "nowrap" }}>{sk.desc}</p>
              </div>
              <div style={{ textAlign: "right", flexShrink: 0 }}>
                <p style={{ color: DS.green, fontWeight: 700, fontSize: 13, margin: "0 0 2px" }}>~{fmtTokens(sk.savings)} saved</p>
                <p style={{ color: "#555553", fontSize: 11, margin: 0 }}>Used {sk.uses}× · {sk.created}</p>
              </div>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}
