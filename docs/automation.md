# ⚙️ Automations — Circular Raw Material Calculator

Este documento descreve a automação recomendada para calcular KPIs de circularidade e scoring ESG a partir da tabela **Materiais**, e atualizar a tabela **Resultados & KPIs** (um registo por empresa).

> Plataforma-alvo: **Airtable Automations** (Run script, JavaScript)  
> Requisitos: plano com *Run script*, campos ligados por **Link to another record** no campo `Empresa`.

---

## 1) Estrutura das Tabelas

- **Diagnóstico ESG & Circularidade**
  - `Empresa` (Primary, texto)
  - `Unidades Produzidas` (número)
  - `Unidades Vendidas` (número)
  - `Peso Unitário (kg)` (número, decimal)
  - … (campos de perfil)

- **Materiais** *(n ≥ 0 por empresa)*
  - `Empresa` (**Link** → Diagnóstico ESG & Circularidade)
  - `Nome do Material` (texto)
  - `Quantidade Entrada (kg)` (número)
  - `% Reciclado na Entrada` (%)  
  - `% Perdas no Processo` (%)  
  - `Resíduos (kg)` (número)
  - `Destino Resíduos` (select: Aterro / Reciclagem interna / Reciclagem externa / Valorização energética)
  - `% Recolha Fim-de-Vida` (%)
  - `Custo de Compra (€/kg)` (número)
  - `Custo de Disposição (€/kg)` (número)
  - `Energia associada (kWh/kg)` (número, opcional)
  - `Risco (1-5)` (número)

- **Checklist ESG** *(m ≥ 0 por empresa)*
  - `Empresa` (**Link** → Diagnóstico ESG & Circularidade)
  - `Pilar` (select: Ambiental / Social / Governança / Turismo)
  - `Pergunta` (texto)
  - `Resposta` (single select / checkbox / número)

- **Resultados & KPIs** *(1 por empresa)*
  - `Empresa` (**Link** → Diagnóstico ESG & Circularidade)
  - `Circularidade na Entrada (%)` (número)
  - `Valorização de Resíduos (%)` (número)
  - `Recolha Fim-de-Vida (%)` (número)
  - `Rácio Resíduo/Produto (kg/un)` (número)
  - `Custo Material / Unidade (€)` (número)
  - `Custo Perdas / Unidade (€)` (número)
  - `Top 3 Hotspots` (texto)
  - `Score ESG E (%)`, `Score ESG S (%)`, `Score ESG G (%)` (número 0–100)
  - `Score ESG Global (%)` (número 0–100)
  - `Faixa ESG` (select: Emergente / Em Desenvolvimento / Avançado / Referência)

> **Dica:** define os campos % como **Percent** no Airtable; os monetários/quantidades como **Number** (decimais).

---

## 2) Trigger (dispara a automação)

Escolhe **um**:

- **When record created** em `Materiais`; ou  
- **When record updated** em `Materiais` (campos críticos: quantidade, %, resíduos, custos, destino).

Na ação **Run script**, adiciona uma **Input variable**:
- `materialRecordId` → tipo **Record (Materiais)**

*(Se preferires disparar por Diagnóstico, usa `diagnosticoRecordId` — o script também suporta.)*

---

## 3) Script (colar no “Run script”)

> Alinha os nomes das tabelas/campos se estiverem diferentes.  
> O script agrega os materiais da empresa, calcula KPIs, faz ranking de hotspots e atualiza/ cria a respetiva linha em **Resultados & KPIs**.

```javascript
/***** CONFIG *****/
const N = {
  tabelas: { diag:"Diagnóstico ESG & Circularidade", mat:"Materiais", esg:"Checklist ESG", res:"Resultados & KPIs" },
  camposDiag: { empresa:"Empresa", unidadesProd:"Unidades Produzidas", unidadesVend:"Unidades Vendidas", pesoUnit:"Peso Unitário (kg)" },
  camposMat: {
    empresa:"Empresa", nome:"Nome do Material", qtd:"Quantidade Entrada (kg)", pctRecIn:"% Reciclado na Entrada",
    pctPerdas:"% Perdas no Processo", residuos:"Resíduos (kg)", destino:"Destino Resíduos", pctRecolha:"% Recolha Fim-de-Vida",
    custoCompra:"Custo de Compra (€/kg)", custoDisp:"Custo de Disposição (€/kg)", energia:"Energia associada (kWh/kg)", risco:"Risco (1-5)"
  },
  camposRes: {
    empresa:"Empresa", circEntrada:"Circularidade na Entrada (%)", valorizacao:"Valorização de Resíduos (%)",
    recolhaFimVida:"Recolha Fim-de-Vida (%)", ratioResiduoUn:"Rácio Resíduo/Produto (kg/un)", custoMatUn:"Custo Material / Unidade (€)",
    custoPerdasUn:"Custo Perdas / Unidade (€)", hotspots:"Top 3 Hotspots",
    scoreE:"Score ESG E (%)", scoreS:"Score ESG S (%)", scoreG:"Score ESG G (%)",
    scoreGlobal:"Score ESG Global (%)", faixa:"Faixa ESG"
  },
  destinosValorizados: new Set(["Reciclagem interna","Reciclagem externa","Valorização energética"]),
};
/***** FIM CONFIG *****/

const { materialRecordId, diagnosticoRecordId } = input.config();

const tDiag = base.getTable(N.tabelas.diag);
const tMat  = base.getTable(N.tabelas.mat);
const tESG  = base.getTable(N.tabelas.esg);
const tRes  = base.getTable(N.tabelas.res);

let empresaRecordId = null;
if (materialRecordId) {
  const matRec = await tMat.selectRecordAsync(materialRecordId);
  if (!matRec) throw new Error("Material não encontrado.");
  const linked = matRec.getCellValue(N.camposMat.empresa);
  if (!linked?.length) throw new Error("Material sem ligação a Empresa.");
  empresaRecordId = linked[0].id;
} else if (diagnosticoRecordId) {
  empresaRecordId = diagnosticoRecordId;
} else { throw new Error("Passe materialRecordId OU diagnosticoRecordId."); }

const empresaRec = await tDiag.selectRecordAsync(empresaRecordId);
if (!empresaRec) throw new Error("Empresa não encontrada.");

const toNum = (v)=> (v==null||v==="")?0:Number(String(v).replace(",","."))||0;
const toPct = (v)=> toNum(v)/100;
const safeDiv = (a,b)=> b>0? a/b : 0;
const faixaESG = (s)=> s<40?"Emergente": s<60?"Em Desenvolvimento": s<80?"Avançado":"Referência";

// Agregar Materiais
const qMat = await tMat.selectRecordsAsync({ fields:Object.values(N.camposMat) });
const mats = qMat.records.filter(r => (r.getCellValue(N.camposMat.empresa)||[]).some(x=>x.id===empresaRecordId));

let totalEntrada=0,totalRecicladoEntrada=0,totalPerdas=0,totalResiduos=0,totalResiduosValorizados=0,totalCustoMat=0,totalCustoPerdas=0;
let somaRecolhaPond=0,massaParaRecolha=0;
const spots=[];

for (const r of mats){
  const nome = r.getCellValueAsString(N.camposMat.nome) || "(Sem nome)";
  const entrada=toNum(r.getCellValue(N.camposMat.qtd));
  const pRecIn=toPct(r.getCellValue(N.camposMat.pctRecIn));
  const pPerd=toPct(r.getCellValue(N.camposMat.pctPerdas));
  const residuos=toNum(r.getCellValue(N.camposMat.residuos));
  const destino=r.getCellValueAsString(N.camposMat.destino);
  const pRecolha=toPct(r.getCellValue(N.camposMat.pctRecolha));
  const cCompra=toNum(r.getCellValue(N.camposMat.custoCompra));
  const cDisp=toNum(r.getCellValue(N.camposMat.custoDisp));
  const risco=toNum(r.getCellValue(N.camposMat.risco));

  const recicladoEntrada=entrada*pRecIn;
  const perdas=entrada*pPerd;
  const custoMat=entrada*cCompra;
  const custoPerdas=(perdas*cCompra)+(residuos*cDisp);

  totalEntrada+=entrada;
  totalRecicladoEntrada+=recicladoEntrada;
  totalPerdas+=perdas;
  totalResiduos+=residuos;
  if (destino && N.destinosValorizados.has(destino)) totalResiduosValorizados+=residuos;
  totalCustoMat+=custoMat;
  totalCustoPerdas+=custoPerdas;

  const massaProduto=Math.max(entrada-perdas,0);
  somaRecolhaPond += massaProduto*pRecolha;
  massaParaRecolha += massaProduto;

  spots.push({nome, perdasKg:perdas, custoPerdas, risco});
}

// KPIs globais
const circEntradaPct = safeDiv(totalRecicladoEntrada,totalEntrada)*100;
const valorizacaoPct = safeDiv(totalResiduosValorizados,totalResiduos)*100;
const recolhaPct     = safeDiv(somaRecolhaPond,massaParaRecolha)*100;

const unidadesProd = toNum(empresaRec.getCellValue(N.camposDiag.unidadesProd));
const pesoUnit     = toNum(empresaRec.getCellValue(N.camposDiag.pesoUnit));
const residuosPorUn = unidadesProd>0 ? safeDiv(totalResiduos,unidadesProd) : 0;
const custoMatPorUn = unidadesProd>0 ? safeDiv(totalCustoMat,unidadesProd) : 0;
const custoPerdasPorUn = unidadesProd>0 ? safeDiv(totalCustoPerdas,unidadesProd) : 0;

// Hotspots
spots.sort((a,b)=> (b.custoPerdas-a.custoPerdas) || (b.perdasKg-a.perdasKg) || (b.risco-a.risco));
const top3 = spots.slice(0,3).map(x=>x.nome).join(", ") || "(sem hotspots)";

// Mini-score ESG (Checklist)
const qESG = await tESG.selectRecordsAsync();
const esgRecs = qESG.records.filter(r => (r.getCellValue("Empresa")||[]).some(x=>x.id===empresaRecordId));
const toScore=(v)=>{ if(v==null) return null; const s=String(v).trim().toLowerCase();
  if(["sim","yes","true"].includes(s)) return 100; if(["não","nao","no","false"].includes(s)) return 0;
  const n=toNum(s.replace("%","")); if(n>0 && n<=5 && !s.includes("%")) return (n/5)*100; if(n>=0 && n<=100) return n; return null; };
const avg=(arr)=> arr.length? arr.reduce((a,b)=>a+b,0)/arr.length : 0;
const pilar=(p)=> avg(esgRecs.filter(r => (r.getCellValueAsString("Pilar")||"").toLowerCase()===p).map(r=>toScore(r.getCellValue("Resposta"))).filter(v=>v!=null));

const scoreE = pilar("ambiental"); const scoreS = pilar("social"); const scoreG = pilar("governança");
const scoreGlobal = (scoreE*0.35 + scoreS*0.30 + scoreG*0.35);
const faixa = faixaESG(scoreGlobal);

// Criar/Atualizar Resultados
const qRes = await tRes.selectRecordsAsync({ fields:[N.camposRes.empresa] });
let resRec = qRes.records.find(r => (r.getCellValue(N.camposRes.empresa)||[]).some(x=>x.id===empresaRecordId));
if(!resRec){
  const id = await tRes.createRecordAsync({ [N.camposRes.empresa]: [{id:empresaRecordId}] });
  resRec = await tRes.selectRecordAsync(id);
}
await tRes.updateRecordAsync(resRec, {
  [N.camposRes.circEntrada]: Number(circEntradaPct.toFixed(2)),
  [N.camposRes.valorizacao]: Number(valorizacaoPct.toFixed(2)),
  [N.camposRes.recolhaFimVida]: Number(recolhaPct.toFixed(2)),
  [N.camposRes.ratioResiduoUn]: Number(residuosPorUn.toFixed(4)),
  [N.camposRes.custoMatUn]: Number(custoMatPorUn.toFixed(4)),
  [N.camposRes.custoPerdasUn]: Number(custoPerdasPorUn.toFixed(4)),
  [N.camposRes.hotspots]: top3,
  [N.camposRes.scoreE]: Math.round(scoreE||0),
  [N.camposRes.scoreS]: Math.round(scoreS||0),
  [N.camposRes.scoreG]: Math.round(scoreG||0),
  [N.camposRes.scoreGlobal]: Math.round(scoreGlobal||0),
  [N.camposRes.faixa]: faixa
});

// Saída p/ debug
output.set("empresa", empresaRec.name);
output.set("materiais_processados", mats.length);
output.set("kpis", { circEntradaPct, valorizacaoPct, recolhaPct, residuosPorUn, custoMatPorUn, custoPerdasPorUn, top3 });
