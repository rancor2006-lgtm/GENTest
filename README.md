import React, { useEffect, useMemo, useRef, useState } from "react";
import * as d3 from "d3";

// Minimal single-file version: no external UI kits, no Tailwind required.
// Uses plain HTML elements and inline styles so it runs in a fresh Vite React app.

/** Minimal genealogy graph app (no-dup, incest-safe)
 * - One node per person, union nodes for relationships
 * - Pan/zoom, add/edit nodes, JSON import/export, SVG export
 * - No external UI libraries required (just React + d3)
 */

// --------------------------- Sample Data ---------------------------
const sampleData = {
  people: [
    { id: "P1", name: "Ptolemy I Soter", sex: "M", notes: "r. 305–282 BC" },
    { id: "P2", name: "Berenice I", sex: "F", notes: "Queen consort" },
    { id: "P3", name: "Ptolemy II Philadelphus", sex: "M", notes: "married sister" },
    { id: "P4", name: "Arsinoe II Philadelphus", sex: "F", notes: "married brother" },
    { id: "P5", name: "Arsinoe I", sex: "F", notes: "1st wife of Ptolemy II" },
    { id: "P6", name: "Ptolemy III Euergetes", sex: "M", notes: "son of P2 & P5" },
    { id: "P7", name: "Berenice II", sex: "F", notes: "queen" },
    { id: "P8", name: "Cleopatra VII Philopator", sex: "F", notes: "queen" },
    { id: "P9", name: "Ptolemy XII Auletes", sex: "M", notes: "father of Cleopatra VII" },
    { id: "P10", name: "Cleopatra V Tryphaena", sex: "F", notes: "consort" },
    { id: "P11", name: "Ptolemy XIII Theos Philopator", sex: "M", notes: "brother/husband" },
    { id: "P12", name: "Ptolemy XIV", sex: "M", notes: "younger brother" },
    { id: "P13", name: "Ptolemy IV Philopator", sex: "M", notes: "son of P6 & P7" },
    { id: "P14", name: "Arsinoe III", sex: "F", notes: "sister-wife of P13" },
  ],
  unions: [
    { id: "U1", partnerA: "P1", partnerB: "P2", notes: "Ptolemy I × Berenice I" },
    { id: "U2", partnerA: "P3", partnerB: "P5", notes: "Ptolemy II × Arsinoe I" },
    { id: "U3", partnerA: "P6", partnerB: "P7", notes: "Ptolemy III × Berenice II" },
    { id: "U4", partnerA: "P3", partnerB: "P4", notes: "Sibling marriage" },
    { id: "U5", partnerA: "P9", partnerB: "P10", notes: "Ptolemy XII × Cleopatra V" },
    { id: "U6", partnerA: "P8", partnerB: "P11", notes: "Sibling marriage" },
    { id: "U7", partnerA: "P13", partnerB: "P14", notes: "Sibling marriage" },
  ],
  childLinks: [
    { unionId: "U1", childId: "P3" },
    { unionId: "U1", childId: "P4" },
    { unionId: "U2", childId: "P6" },
    { unionId: "U3", childId: "P13" },
    { unionId: "U3", childId: "P14" },
    { unionId: "U5", childId: "P8" },
    { unionId: "U5", childId: "P11" },
    { unionId: "U5", childId: "P12" },
  ],
};

// --------------------------- Utilities ---------------------------
const STORAGE_KEY = "genealogy-graph-v1";

function useLocalStorageState(key, initial) {
  const [state, setState] = useState(() => {
    const raw = typeof window !== "undefined" ? localStorage.getItem(key) : null;
    return raw ? JSON.parse(raw) : initial;
  });
  useEffect(() => {
    try { localStorage.setItem(key, JSON.stringify(state)); } catch {}
  }, [key, state]);
  return [state, setState];
}

function downloadText(filename, text) {
  const blob = new Blob([text], { type: "application/json" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url; a.download = filename; a.click();
  URL.revokeObjectURL(url);
}

function readFileAsText(file) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => resolve(reader.result);
    reader.onerror = reject;
    reader.readAsText(file);
  });
}

// --------------------------- Graph Builder ---------------------------
function buildGraph(data) {
  const peopleMap = new Map(data.people.map(p => [p.id, p]));

  // Nodes: persons + unions
  const nodes = [
    ...data.people.map(p => ({
      id: p.id,
      type: "person",
      name: p.name,
      sex: p.sex || "?",
      notes: p.notes || "",
      fx: undefined, fy: undefined,
    })),
    ...data.unions.map(u => ({
      id: u.id,
      type: "union",
      partnerA: u.partnerA,
      partnerB: u.partnerB,
      notes: u.notes || "",
    })),
  ];

  // Links: partner→union (no arrow), union→child (arrow)
  const links = [];
  for (const u of data.unions) {
    links.push({ source: u.partnerA, target: u.id, kind: "partner" });
    links.push({ source: u.partnerB, target: u.id, kind: "partner" });
  }
  for (const cu of data.childLinks) {
    links.push({ source: cu.unionId, target: cu.childId, kind: "child" });
  }

  return { nodes, links };
}

// --------------------------- Graph View ---------------------------
function GraphView({ data, settings, selectedId, setSelectedId, highlightText }) {
  const ref = useRef(null);
  const svgRef = useRef(null);
  const { nodes, links } = useMemo(() => buildGraph(data), [data]);

  useEffect(() => {
    if (!ref.current) return;

    const width = ref.current.clientWidth;
    const height = ref.current.clientHeight;

    const svg = d3.select(svgRef.current)
      .attr("viewBox", [0, 0, width, height])
      .call(d3.zoom().scaleExtent([0.1, 3]).on("zoom", (event) => {
        g.attr("transform", event.transform);
      }));

    svg.selectAll("*").remove();
    const g = svg.append("g");

    const link = g.append("g").attr("stroke", "#888").attr("stroke-opacity", 0.7)
      .selectAll("line").data(links).join("line")
      .attr("stroke-width", d => d.kind === "partner" ? 1.6 : 1.2)
      .attr("marker-end", d => d.kind === "child" ? "url(#arrow)" : null);

    const defs = svg.append("defs");
    defs.append("marker")
      .attr("id", "arrow")
      .attr("viewBox", "0 -5 10 10")
      .attr("refX", 14)
      .attr("refY", 0)
      .attr("markerWidth", 6)
      .attr("markerHeight", 6)
      .attr("orient", "auto")
      .append("path")
      .attr("d", "M0,-5L10,0L0,5")
      .attr("fill", "#888");

    const drag = d3.drag()
      .on("start", (event, d) => {
        if (!event.active) sim.alphaTarget(0.3).restart();
        d.fx = d.x; d.fy = d.y;
      })
      .on("drag", (event, d) => {
        d.fx = event.x; d.fy = event.y;
      })
      .on("end", (event, d) => {
        if (!event.active) sim.alphaTarget(0);
        if (!settings.lockOnDragEnd) { d.fx = null; d.fy = null; }
      });

    const node = g.append("g").selectAll("g").data(nodes).join("g")
      .attr("cursor", "pointer")
      .call(drag)
      .on("click", (_, d) => setSelectedId(d.id));

    // Node shapes
    node.append("path")
      .attr("d", d => d.type === "union" ? d3.symbol().type(d3.symbolDiamond).size(90)() : d.sex === "M" ? roundedRect(36, 22, 4) : ellipsePath(20, 14))
      .attr("fill", d => d.type === "union" ? "#9ca3af" : d.sex === "M" ? "#dbeafe" : d.sex === "F" ? "#fde7f3" : "#f3f4f6")
      .attr("stroke", d => selectedId === d.id ? "#0ea5e9" : "#4b5563")
      .attr("stroke-width", d => selectedId === d.id ? 2.4 : 1.2);

    // Labels
    if (settings.showLabels) {
      node.append("text")
        .attr("y", 28)
        .attr("text-anchor", "middle")
        .attr("font-size", 11)
        .attr("fill", "#111827")
        .text(d => d.type === "union" ? "" : labelFor(d, highlightText));

      if (settings.showNotes) {
        node.append("text")
          .attr("y", 40)
          .attr("text-anchor", "middle")
          .attr("font-size", 9)
          .attr("fill", "#6b7280")
          .text(d => d.type === "union" ? (d.notes || "") : (d.notes || ""));
      }
    }

    const sim = d3.forceSimulation(nodes)
      .force("link", d3.forceLink(links).id(d => d.id).distance(settings.linkDistance).strength(0.6))
      .force("charge", d3.forceManyBody().strength(settings.charge))
      .force("collide", d3.forceCollide().radius(settings.collideRadius))
      .force("center", d3.forceCenter(width/2, height/2))
      .on("tick", ticked);

    function ticked() {
      link
        .attr("x1", d => d.source.x)
        .attr("y1", d => d.source.y)
        .attr("x2", d => d.target.x)
        .attr("y2", d => d.target.y);

      node.attr("transform", d => `translate(${d.x},${d.y})`);
    }

    function labelFor(d, highlight) {
      const name = d.name || d.id;
      if (!highlight) return name;
      const idx = name.toLowerCase().indexOf(highlight.toLowerCase());
      if (idx === -1) return name;
      return name.slice(0, idx) + "[" + name.slice(idx, idx + highlight.length) + "]" + name.slice(idx + highlight.length);
    }

    function roundedRect(w, h, r) {
      const x = -w/2, y = -h/2;
      return `M${x+r},${y}h${w-2*r}a${r},${r} 0 0 1 ${r},${r}v${h-2*r}a${r},${r} 0 0 1 -${r},${r}h-${w-2*r}a${r},${r} 0 0 1 -${r},-${r}v-${h-2*r}a${r},${r} 0 0 1 ${r},-${r}Z`;
    }
    function ellipsePath(rx, ry) {
      return `M ${-rx},0 a ${rx},${ry} 0 1,0 ${2*rx},0 a ${rx},${ry} 0 1,0 -${2*rx},0`;
    }

    return () => { sim.stop(); };
  }, [data, settings, selectedId, setSelectedId, highlightText]);

  return (
    <div ref={ref} style={{ width: "100%", height: "100%" }}>
      <svg ref={svgRef} style={{ width: "100%", height: "100%", background: "#fff", borderRadius: 12, boxShadow: "0 2px 12px rgba(0,0,0,0.08)" }} />
    </div>
  );
}

// --------------------------- Editor Panel ---------------------------
function LabeledInput({ label, value, onChange, placeholder }) {
  return (
    <label style={{ display: "grid", gridTemplateColumns: "120px 1fr", gap: 8, alignItems: "center", fontSize: 12 }}>
      <span>{label}</span>
      <input value={value} onChange={onChange} placeholder={placeholder} style={{ padding: "6px 8px", border: "1px solid #d1d5db", borderRadius: 6 }} />
    </label>
  );
}

function Editor({ data, setData, selectedId, setSelectedId }) {
  const [person, setPerson] = useState({ id: "", name: "", sex: "?", notes: "" });
  const [union, setUnion] = useState({ id: "", partnerA: "", partnerB: "", notes: "" });
  const [child, setChild] = useState({ unionId: "", childId: "" });

  const ids = new Set([...data.people.map(p=>p.id), ...data.unions.map(u=>u.id)]);

  function addPerson() {
    if (!person.id || ids.has(person.id)) return alert("Person ID required and must be unique.");
    setData(d => ({ ...d, people: [...d.people, person] }));
    setPerson({ id: "", name: "", sex: "?", notes: "" });
  }
  function addUnion() {
    if (!union.id || ids.has(union.id)) return alert("Union ID required and must be unique.");
    if (!union.partnerA || !union.partnerB) return alert("Both partners required (person IDs).");
    setData(d => ({ ...d, unions: [...d.unions, union] }));
    setUnion({ id: "", partnerA: "", partnerB: "", notes: "" });
  }
  function addChildLink() {
    if (!child.unionId || !child.childId) return alert("Union ID and Child ID required.");
    setData(d => ({ ...d, childLinks: [...d.childLinks, child] }));
    setChild({ unionId: "", childId: "" });
  }
  function removeSelected() {
    if (!selectedId) return;
    setData(d => {
      const isPerson = d.people.some(p=>p.id===selectedId);
      if (isPerson) {
        const usedAsPartner = d.unions.some(u => u.partnerA===selectedId || u.partnerB===selectedId);
        const usedAsChild = d.childLinks.some(c => c.childId===selectedId);
        if (usedAsPartner || usedAsChild) { alert("Cannot delete: detach from unions/children first."); return d; }
        return { ...d, people: d.people.filter(p=>p.id!==selectedId) };
      } else {
        return { ...d, unions: d.unions.filter(u=>u.id!==selectedId), childLinks: d.childLinks.filter(c=>c.unionId!==selectedId) };
      }
    });
    setSelectedId(null);
  }

  const groupStyle = { border: "1px solid #e5e7eb", borderRadius: 10, padding: 12, display: "grid", gap: 8 };
  const button = (label, onClick, kind="primary") => (
    <button onClick={onClick} style={{
      padding: "6px 10px", borderRadius: 8, border: "1px solid #d1d5db",
      background: kind==="danger"? "#fee2e2" : "#f3f4f6", cursor: "pointer"
    }}>{label}</button>
  );

  return (
    <div style={{ display: "grid", gap: 12 }}>
      <div style={groupStyle}>
        <strong>Add Person</strong>
        <LabeledInput label="ID" value={person.id} onChange={e=>setPerson({...person, id:e.target.value})} placeholder="P101"/>
        <LabeledInput label="Name" value={person.name} onChange={e=>setPerson({...person, name:e.target.value})} placeholder="Name"/>
        <LabeledInput label="Sex" value={person.sex} onChange={e=>setPerson({...person, sex:e.target.value})} placeholder="M/F/?"/>
        <LabeledInput label="Notes" value={person.notes} onChange={e=>setPerson({...person, notes:e.target.value})} placeholder="Details"/>
        {button("Add Person", addPerson)}
      </div>

      <div style={groupStyle}>
        <strong>Add Union</strong>
        <LabeledInput label="Union ID" value={union.id} onChange={e=>setUnion({...union, id:e.target.value})} placeholder="U201"/>
        <LabeledInput label="Partner A" value={union.partnerA} onChange={e=>setUnion({...union, partnerA:e.target.value})} placeholder="Person ID"/>
        <LabeledInput label="Partner B" value={union.partnerB} onChange={e=>setUnion({...union, partnerB:e.target.value})} placeholder="Person ID"/>
        <LabeledInput label="Notes" value={union.notes} onChange={e=>setUnion({...union, notes:e.target.value})} placeholder="Context"/>
        {button("Add Union", addUnion)}
      </div>

      <div style={groupStyle}>
        <strong>Add Child Link</strong>
        <LabeledInput label="Union ID" value={child.unionId} onChange={e=>setChild({...child, unionId:e.target.value})} placeholder="U201"/>
        <LabeledInput label="Child ID" value={child.childId} onChange={e=>setChild({...child, childId:e.target.value})} placeholder="P303"/>
        {button("Add Link", addChildLink)}
      </div>

      <div style={groupStyle}>
        <strong>Danger</strong>
        <div style={{ display:"flex", alignItems:"center", justifyContent:"space-between" }}>
          <span style={{ fontFamily:"monospace" }}>Selected: {selectedId || "(none)"}</span>
          {button("Delete Selected", removeSelected, "danger")}
        </div>
      </div>
    </div>
  );
}

// --------------------------- Main App ---------------------------
export default function GenealogyGraphApp() {
  const [data, setData] = useLocalStorageState(STORAGE_KEY, sampleData);
  const [settings, setSettings] = useState({
    showLabels: true,
    showNotes: true,
    linkDistance: 70,
    charge: -140,
    collideRadius: 22,
    lockOnDragEnd: false,
  });
  const [selectedId, setSelectedId] = useState(null);
  const [search, setSearch] = useState("");

  function resetToSample() { setData(sampleData); setSelectedId(null); }
  function exportJSON() { downloadText("genealogy-graph.json", JSON.stringify(data, null, 2)); }
  async function importJSON(ev) {
    const f = ev.target.files?.[0]; if (!f) return;
    try {
      const text = await readFileAsText(f);
      const obj = JSON.parse(text);
      if (!obj.people || !obj.unions || !obj.childLinks) throw new Error("Invalid format");
      setData(obj); setSelectedId(null);
    } catch (e) {
      alert("Import failed: " + e.message);
    } finally {
      ev.target.value = "";
    }
  }

  function exportSVG() {
    const svg = document.querySelector("svg");
    if (!svg) return;
    const svgData = new XMLSerializer().serializeToString(svg);
    downloadText("genealogy-graph.svg", svgData);
  }

  return (
    <div className="w-full h-screen grid grid-cols-12 gap-4 p-4 bg-gray-50">
      <div className="col-span-9 flex flex-col gap-3">
        <div className="flex items-center gap-2">
          <Button onClick={resetToSample} variant="secondary"><RefreshCw className="w-4 h-4 mr-1"/>Demo</Button>
          <label className="inline-flex items-center gap-2">
            <Button asChild>
              <span className="inline-flex items-center"><Upload className="w-4 h-4 mr-1"/>Import JSON</span>
            </Button>
            <input type="file" accept="application/json" className="hidden" onChange={importJSON} />
          </label>
          <Button onClick={exportJSON}><Save className="w-4 h-4 mr-1"/>Export JSON</Button>
          <Button onClick={exportSVG}><Download className="w-4 h-4 mr-1"/>Export SVG</Button>
          <div className="ml-auto flex items-center gap-2">
            <Search className="w-4 h-4"/>
            <Input placeholder="Find name…" value={search} onChange={e=>setSearch(e.target.value)} className="w-56"/>
          </div>
        </div>
        <div className="flex items-center gap-6 bg-white rounded-2xl p-3 shadow">
          <div className="flex items-center gap-2">
            <Switch checked={settings.showLabels} onCheckedChange={v=>setSettings(s=>({...s, showLabels:v}))} />
            <Label>Labels</Label>
          </div>
          <div className="flex items-center gap-2">
            <Switch checked={settings.showNotes} onCheckedChange={v=>setSettings(s=>({...s, showNotes:v}))} />
            <Label>Notes</Label>
          </div>
          <div className="flex items-center gap-2 w-64">
            <Label className="w-28">Link dist</Label>
            <Slider value={[settings.linkDistance]} min={30} max={180} step={5} onValueChange={([v])=>setSettings(s=>({...s, linkDistance:v}))} />
          </div>
          <div className="flex items-center gap-2 w-64">
            <Label className="w-28">Charge</Label>
            <Slider value={[settings.charge]} min={-400} max={100} step={10} onValueChange={([v])=>setSettings(s=>({...s, charge:v}))} />
          </div>
          <div className="flex items-center gap-2 w-64">
            <Label className="w-28">Collide</Label>
            <Slider value={[settings.collideRadius]} min={10} max={60} step={1} onValueChange={([v])=>setSettings(s=>({...s, collideRadius:v}))} />
          </div>
          <div className="flex items-center gap-2">
            <Switch checked={settings.lockOnDragEnd} onCheckedChange={v=>setSettings(s=>({...s, lockOnDragEnd:v}))} />
            <Label>Lock on drag end</Label>
          </div>
        </div>
        <div className="flex-1 min-h-0">
          <GraphView data={data} settings={settings} selectedId={selectedId} setSelectedId={setSelectedId} highlightText={search} />
        </div>
      </div>
      <div className="col-span-3 flex flex-col gap-3">
        <Card className="h-full">
          <CardHeader>
            <CardTitle>Editor</CardTitle>
          </CardHeader>
          <CardContent className="space-y-4">
            <Editor data={data} setData={setData} selectedId={selectedId} setSelectedId={setSelectedId} />
            <div className="text-xs text-gray-500 leading-snug">
              <p><strong>Tip:</strong> Individuals are <em>never duplicated</em>. Use Unions (diamonds) to connect partners to children. Incest and cousin marriages are naturally represented as loops.</p>
              <p className="mt-2">To keep historical rigor, enforce stable IDs. You can store regnal years, house names, and sources in the notes field and export JSON/SVG for publications.</p>
            </div>
          </CardContent>
        </Card>
      </div>
    </div>
  );
}
