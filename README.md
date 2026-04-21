# ts-morph-to-repo-map

ây đúng là 2 giai đoạn riêng biệt và việc tách chúng ra là best practice cho AI workflow:
Giai đoạn
Nhiệm vụ
Công nghệ
Output
Stage 1
Trích xuất trung thực từ source code
ts-morph + TypeScript Compiler API
raw_project_data.json (đầy đủ, chưa lọc, giữ nguyên cấu trúc AST)
Stage 2
Chuyển đổi, tối ưu token, trích ngữ cảnh framework
Pure TS/JS (JSON transform)
repo_map.json (chuẩn AI, gọn, có decorator context, dependency graph)
Dưới đây là code production-ready cho cả 2 stage, tích hợp đầy đủ xử lý decorator (NestJS/TypeORM), path resolution, và tối ưu token.
📦 Stage 1: extract-raw.ts (ts-morph → raw JSON)
Chỉ làm 1 việc: đọc project, trích xuất mọi symbol/import/export trung thực, không filter, không transform.
// extract-raw.ts
import { Project, Node } from "ts-morph";
import * as fs from "fs";
import * as path from "path";
const ROOT = process.cwd();
const OUT = path.join(ROOT, "raw_project_data.json");
const project = new Project({
tsConfigFilePath: path.join(ROOT, "tsconfig.json"),
compilerOptions: { skipLibCheck: true, noEmit: true },
});
function safeType(node: Node) {
try {
return node.getTypeNode()?.getText() ?? node.getType().getText(node);
} catch {
return "unknown";
}
}
function extractDecoratorsRaw(node: Node) {
if (!("getDecorators" in node)) return [];
return (node as any).getDecorators().map((d: any) => ({
name: d.getName(),
args: d.getArguments().map((a: any) => a.getText()),
}));
}
const rawFiles: Record<string, any> = {};
for (const sf of project.getSourceFiles()) {
const fp = sf.getFilePath();
if (fp.includes("node_modules") || /.(test|spec)./.test(fp)) continue;
const rel = path.relative(ROOT, fp).replace(/\/g, "/");
const symbols: any[] = [];
// Functions
for (const fn of sf.getFunctions()) {
symbols.push({
kind: "function",
name: fn.getName() ?? "anonymous",
exported: fn.isExported(),
async: fn.isAsync(),
params: fn.getParameters().map(p => ({ name: p.getName(), type: safeType(p) })),
returns: safeType(fn),
doc: fn.getJsDocs()[0]?.getDescription()?.trim().split("\n")[0] ?? null,
decorators: extractDecoratorsRaw(fn),
});
}
// Classes
for (const cls of sf.getClasses()) {
symbols.push({
kind: "class",
name: cls.getName() ?? "anonymous",
exported: cls.isExported(),
extends: cls.getExtends()?.getExpression().getText() ?? null,
implements: cls.getImplements().map(i => i.getExpression().getText()),
decorators: extractDecoratorsRaw(cls),
methods: cls.getMethods().map(m => ({
name: m.getName(),
scope: m.getScope() ?? "public",
async: m.isAsync(),
params: m.getParameters().map(p => ({ name: p.getName(), type: safeType(p) })),
returns: safeType(m),
doc: m.getJsDocs()[0]?.getDescription()?.trim().split("\n")[0] ?? null,
decorators: extractDecoratorsRaw(m),
})),
properties: cls.getProperties().map(p => ({
name: p.getName(),
type: safeType(p),
scope: p.getScope() ?? "public",
decorators: extractDecoratorsRaw(p),
})),
});
}
// Interfaces
for (const iface of sf.getInterfaces()) {
symbols.push({
kind: "interface",
name: iface.getName(),
exported: iface.isExported(),
extends: iface.getExtends().map(e => e.getExpression().getText()),
members: iface.getProperties().map(p => ({
name: p.getName(),
type: safeType(p),
optional: p.hasQuestionToken(),
})),
});
}
// Type Aliases
for (const ta of sf.getTypeAliases()) {
symbols.push({
kind: "type",
name: ta.getName(),
exported: ta.isExported(),
definition: safeType(ta),
});
}
// Enums
for (const en of sf.getEnums()) {
symbols.push({
kind: "enum",
name: en.getName(),
exported: en.isExported(),
members: en.getMembers().map(m => m.getName()),
});
}
// Variables
for (const stmt of sf.getVariableStatements()) {
for (const decl of stmt.getDeclarations()) {
symbols.push({
kind: "variable",
name: decl.getName(),
exported: stmt.isExported(),
type: safeType(decl),
doc: stmt.getJsDocs()[0]?.getDescription()?.trim().split("\n")[0] ?? null,
});
}
}
// Imports
const imports = sf.getImportDeclarations().map(imp => ({
specifier: imp.getModuleSpecifierValue(),
resolvedPath: imp.getModuleSpecifierSourceFile()?.getFilePath() ?? null,
named: imp.getNamedImports().map(n => n.getName()),
default: imp.getDefaultImport()?.getText() ?? null,
}));
rawFiles[rel] = { imports, symbols };
}
fs.writeFileSync(OUT, JSON.stringify({ version: 1, generated: new Date().toISOString(), files: rawFiles }, null, 2));
console.log(✅ Stage 1 done: ${Object.keys(rawFiles).length} files → raw_project_data.json);
🔄 Stage 2: transform-to-repo-map.ts (raw JSON → AI Repo Map)
Đọc raw JSON, lọc noise, parse decorator framework, resolve path, tối ưu token, xuất schema chuẩn AI.
// transform-to-repo-map-md.ts
import * as fs from "fs";
import * as path from "path";
const ROOT = process.cwd();
const RAW = path.join(ROOT, "raw_project_data.json");
const OUT = path.join(ROOT, "REPO_MAP.md");
if (!fs.existsSync(RAW)) {
console.error("❌ raw_project_data.json not found. Run extract-raw.ts first.");
process.exit(1);
}
const raw = JSON.parse(fs.readFileSync(RAW, "utf-8"));
const rawFiles = raw.files as Record<string, any>;
// ─────────────────────────────────────────────────────────────
// 🔍 1. BUILD GRAPH & DETECT CYCLES
// ─────────────────────────────────────────────────────────────
const graph: Record<string, Set<string>> = {};
const allFiles = Object.keys(rawFiles);
for (const f of allFiles) graph[f] = new Set();
for (const [filePath, fileData] of Object.entries(rawFiles)) {
for (const imp of fileData.imports || []) {
if (imp.internal) {
const target = imp.resolvedPath
? path.relative(ROOT, imp.resolvedPath).replace(/\/g, "/")
: imp.from;
if (graph[target]) graph[filePath].add(target);
}
}
}
const cycles: string[][] = [];
const state = new Map<string, number>();
const dfsPath: string[] = [];
function findCycles(node: string) {
if (state.get(node) === 2) return;
if (state.get(node) === 1) {
const idx = dfsPath.indexOf(node);
if (idx !== -1) {
const cycle = dfsPath.slice(idx);
cycle.push(node);
cycles.push(cycle);
}
return;
}
state.set(node, 1);
dfsPath.push(node);
for (const nb of graph[node] || []) findCycles(nb);
dfsPath.pop();
state.set(node, 2);
}
for (const node of allFiles) if (!state.has(node)) findCycles(node);
// Chuẩn hóa & khử trùng
const uniqueCycles = new Set<string>();
const normalizedCycles: string[][] = [];
for (const c of cycles) {
const loop = c.slice(0, -1);
const minFile = loop.reduce((a, b) => (a < b ? a : b));
const minIdx = loop.indexOf(minFile);
const normalized = [...loop.slice(minIdx), ...loop.slice(0, minIdx)];
const key = normalized.join(" → ");
if (!uniqueCycles.has(key)) {
uniqueCycles.add(key);
normalizedCycles.push([...normalized, normalized[0]]);
}
}
// ─────────────────────────────────────────────────────────────
// 🧠 2. CIRCULAR DEPENDENCY ANALYZER & SUGGESTION ENGINE
// ─────────────────────────────────────────────────────────────
function normalizePath(p: string) {
return p.replace(/\/g, "/").replace(/.tsx?
/, "");
}
function analyzeCycleAndSuggest(cycle: string[]): string {
const nodes = cycle.slice(0, -1);
const suggestions: string[] = [];
const crossImports: { from: string; to: string; symbols: string[]; kinds: string[] }[] = [];
// Thu thập import chéo trong chu trình
for (let i = 0; i < nodes.length; i++) {
const current = nodes[i];
const next = nodes[(i + 1) % nodes.length];
const fileData = rawFiles[current];
if (!fileData?.imports) continue;
code
Code
for (const imp of fileData.imports) {
  const impTarget = normalizePath(imp.resolvedPath || imp.from);
  const nextNorm = normalizePath(next);
  if (impTarget === nextNorm) {
    const symbols = imp.names || [];
    const kinds: string[] = [];
    const targetData = rawFiles[next];
    if (targetData?.symbols) {
      for (const sym of symbols) {
        const cleanSym = sym.replace(/^default:/, "");
        const found = targetData.symbols.find((s: any) => s.name === cleanSym);
        if (found) kinds.push(found.kind);
      }
    }
    crossImports.push({ from: current, to: next, symbols, kinds });
  }
}
}
// 1. Phát hiện barrel file
const barrels = nodes.filter(f => f.endsWith("index.ts") || f.endsWith("index.js"));
if (barrels.length > 0) {
suggestions.push(🔹 Barrel file tham gia chu trình: \${barrels.join(", ")}`. Ưu tiên import trực tiếp từ file nguồn thay vì qua barrel để tránh implicit cycle.`);
}
// 2. Phân loại symbol bị import chéo
const typeSyms = new Set<string>();
const classSyms = new Set<string>();
const fnSyms = new Set<string>();
for (const ci of crossImports) {
for (let i = 0; i < ci.symbols.length; i++) {
const kind = ci.kinds[i];
const sym = ci.symbols[i];
if (kind === "type" || kind === "interface") typeSyms.add(sym);
else if (kind === "class") classSyms.add(sym);
else if (kind === "function") fnSyms.add(sym);
}
}
if (typeSyms.size > 0) {
suggestions.push(🔹 Tách types/interfaces chung ra file riêng (vd: \shared/types.ts` hoặc `common.interfaces.ts`): ${[...typeSyms].join(", ")}`);
}
if (classSyms.size > 0) {
suggestions.push(🔹 Class coupling detected. Chiến lược phá vỡ: - Dùng Dependency Injection (truyền instance qua constructor) - Định nghĩa interface ở file chung, import interface thay vì class cụ thể - Nếu bắt buộc, dùng lazy import: \const Mod = await import('./path')``);
}
if (fnSyms.size > 0) {
suggestions.push(🔹 Function cross-imports. Nên di chuyển utility functions sang module \utils/` hoặc `helpers/` không có internal dependency.`);
}
if (suggestions.length === 0) {
suggestions.push(🔹 Không tìm thấy boundary rõ ràng. Cân nhắc tái cấu trúc module hoặc引入 mediator/service layer để giảm coupling.);
}
return suggestions.map(s => ${s}).join("\n");
}
// ─────────────────────────────────────────────────────────────
// 📝 3. HELPERS & RENDER
// ─────────────────────────────────────────────────────────────
function parseDecoratorContext(decorators: any[]) {
const ctx: string[] = [];
if (!decorators) return null;
for (const d of decorators) {
const name = d.name;
const args = d.args?.join(" ") ?? "";
if (name === "Controller") ctx.push(@Controller(${args}));
if (["Get", "Post", "Put", "Patch", "Delete"].includes(name)) {
const route = args.match(/['"]([^'"]+)['"]/)?.[1] ?? ""; ctx.push(@
{route}')); } if (name === "Injectable") ctx.push("@Injectable()"); if (name === "Entity") ctx.push(@Entity(
{name}()); if (["OneToMany", "ManyToOne", "OneToOne", "ManyToMany"].includes(name)) ctx.push(@${name}()`);
}
return ctx.length > 0 ? ctx.join(" ") : null;
}
function cleanType(t: string) {
if (!t) return "any";
return t.replace(/import(["'][^"']+["'])./g, "").replace(/\s+/g, " ").trim().slice(0, 150);
}
// Group files by directory
const groupedFiles: Record<string, any[]> = {};
for (const [filePath, fileData] of Object.entries(rawFiles)) {
const dir = path.dirname(filePath);
if (!groupedFiles[dir]) groupedFiles[dir] = [];
groupedFiles[dir].push({ path: filePath, ...fileData });
}
let md = "# 🗺️ Project Repository Map\n\n";
md += > Generated: ${new Date().toISOString()}\n;
md += > Total Files: ${allFiles.length}\n;
md += > Circular Dependencies: ${normalizedCycles.length > 0 ?⚠️ ${normalizedCycles.length} detected: "✅ None"}\n\n;
// ⚠️ Circular Warning + Auto Suggestions
if (normalizedCycles.length > 0) {
md += "## ⚠️ Circular Dependencies Detected\n\n";
md += "Các chu trình dưới đây có thể gây lỗi runtime, khó tree-shake hoặc khởi tạo module sai thứ tự.\n\n";
for (let i = 0; i < normalizedCycles.length; i++) {
md += ### 🔁 Cycle ${i + 1}: \${normalizedCycles[i].join(" → ")}`\n\n`;
md += "💡 Auto-Suggested Fix Strategy:\n";
md += analyzeCycleAndSuggest(normalizedCycles[i]) + "\n\n";
}
md += "> 📌 General Rule: Extract shared contracts (types/interfaces) → Apply DI for classes → Use lazy imports as last resort.\n\n";
md += "---\n\n";
}
// Directory & File Structure
const sortedDirs = Object.keys(groupedFiles).sort();
for (const dir of sortedDirs) {
const dirLabel = dir === "." ? "📂 Root" : 📂 ${dir}/;
md += ### ${dirLabel}\n\n;
const files = groupedFiles[dir].sort((a, b) => a.path.localeCompare(b.path));
for (const file of files) {
const fileName = path.basename(file.path);
md += #### 📄 \${fileName}`\n\n`;
code
Code
if (file.imports?.length > 0) {
  md += `- **Imports:**\n`;
  for (const imp of file.imports) {
    const source = imp.internal ? `\`${imp.from}\`` : imp.from;
    const names = imp.names?.join(", ") || "";
    if (names) md += `  - \`from ${source}\`: ${names}\n`;
  }
  md += "\n";
}

if (file.symbols?.length > 0) {
  md += `- **API Surface:**\n`;
  for (const sym of file.symbols) {
    if (!sym.exported) continue;
    let line = `  - `;
    if (sym.kind === "class") line += "🏛️ ";
    else if (sym.kind === "interface") line += "🧩 ";
    else if (sym.kind === "type") line += "🔷 ";
    else if (sym.kind === "enum") line += "🔢 ";
    else if (sym.kind === "function") line += "⚡ ";
    else if (sym.kind === "variable") line += "📦 ";

    line += `\`${sym.name}\``;

    if (sym.kind === "function") {
      const params = sym.params?.map((p: any) => `${p.name}: ${cleanType(p.type)}`).join(", ") || "";
      line += `(${params}) => ${cleanType(sym.returns)}`;
      if (sym.async) line += " *(async)*";
    } else if (sym.kind === "class") {
      if (sym.extends) line += ` extends \`${sym.extends}\``;
      if (sym.implements?.length) line += ` implements [${sym.implements.join(", ")}]`;
    } else if (sym.kind === "interface") {
      if (sym.extends?.length) line += ` extends [${sym.extends.join(", ")}]`;
      line += ` { ${sym.members?.length || 0} props }`;
    } else if (sym.kind === "type") {
      line += ` = ${cleanType(sym.definition)}`;
    } else if (sym.kind === "enum") {
      line += ` { ${sym.members?.join(", ")} }`;
    } else if (sym.kind === "variable") {
      line += `: ${cleanType(sym.type)}`;
    }

    const decCtx = parseDecoratorContext(sym.decorators);
    if (decCtx) line += ` <span style="color:gray">[${decCtx}]</span>`;
    if (sym.doc) line += ` _"${sym.doc}"_`;

    md += line + "\n";

    if (sym.kind === "class" && sym.methods?.length > 0) {
      const pubMethods = sym.methods.filter((m: any) => m.scope !== "private");
      if (pubMethods.length > 0) {
        md += `    - Methods: ${pubMethods.map((m: any) => `\`${m.name}()\``).join(", ")}\n`;
      }
    }
  }
  md += "\n";
}
md += "---\n";
}
}
md += `
🤖 AI Usage Rules
Locate: Use exact file paths + export names. Never guess paths.
Dependencies: Check `Imports` section. If a file is part of a circular cycle, avoid adding new cross-imports to it.
Framework Context: Tags like `@Controller`, `@Entity` are architectural contracts. Respect them.
No Implementation: This map contains signatures only. Request full file content before editing logic.
Type Safety: If a type shows `unknown` or is truncated, ask for clarification instead of assuming.
`;
fs.writeFileSync(OUT, md);
console.log(✅ REPO_MAP.md generated (${allFiles.length} files, ${normalizedCycles.length} circular deps analyzed));
Vẫn giữ Stage 1 ra JSON: Vì JSON là nguồn sự thật (source of truth), dễ debug, và dễ chuyển đổi sang bất kỳ format nào khác sau này (YAML, XML, v.v.).
Stage 2 xuất ra cả 2:
repo_map.json: Dùng cho các script tự động, tooling nội bộ.
REPO_MAP.md: Dùng để copy-paste vào ChatGPT/Claude/Cursor.
🔍 Engine phân tích chu trình hoạt động thế nào?
Bước
Logic
Kết quả
Path Normalization
Chuẩn hóa đường dẫn: bỏ .ts, /index, \ → /
src/user/index.ts và src/user được coi là cùng 1 node
Cross-Import Extraction
Duyệt từng cạnh trong chu trình, tìm chính xác symbol nào đang bị import chéo
Biết được A import UserService từ B, B import AuthDTO từ A
Kind Classification
So khớp symbol với rawFiles[target].symbols để lấy kind
Phân loại: type/interface, class, function, variable
Strategy Mapping
Ánh xạ kind → chiến lược refactor chuẩn TS/Node
Type → extract file; Class → DI/Interface; Fn → utils; Barrel → explicit import
Dedup & Format
Gộp symbol trùng, format thành bullet points ngắn gọn
Output gọn, AI đọc được ngay, không phình token
🚀 Cách chạy & tích hợp CI/Dev
json
123456789
Chạy: npm run gen:repo-map → tạo raw_project_data.json → transform → repo_map.json
🤖 Cách AI sẽ dùng phần này
Khi bạn paste REPO_MAP.md vào LLM, prompt hệ thống đã được thiết kế để AI:
Đọc cảnh báo trước → tự động tránh đề xuất import làm xoắn thêm chu trình
Áp dụng đúng chiến lược theo loại symbol (không đề xuất DI cho type, không đề xuất extract file cho class service)
Refactor có chủ đích → AI sẽ hỏi: "Bạn có muốn tôi tạo file shared/auth.types.ts và di chuyển AuthPayload, UserDTO sang đó không?" thay vì đoán mò
Cảnh báo barrel → AI hiểu index.ts trong chu trình là anti-pattern và đề xuất explicit import
✅ Checklist trước khi chạy
Đã chạy tsx extract-raw.ts thành công
tsconfig.json có baseUrl/paths nếu dùng alias @/
Repo > 800 files → thêm NODE_OPTIONS="--max-old-space-size=4096"
Kiểm tra REPO_MAP.md → phần ⚠️ Circular Dependencies hiển thị đúng chu trình + gợi ý
